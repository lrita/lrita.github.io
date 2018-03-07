---
layout: post
title: steemd 源码分析1 JSON RPC机制
categories: [c++, steem, blockchain]
description: c++ steem blockchain
keywords: c++ steemd steem blockchain
---

`steemd`通过[`JSON-RPC 2.0`](http://wiki.geekdream.com/Specification/json-rpc_2.0.html)对外提供API调用，
本篇主要分析`steemd`通过何种机制将各个类中的方法变成`JSON-RPC`的api的。

注意：
`steemd`将`JSON-RPC 2.0`做了一个[特殊转换](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L215-L229)
（可能是兼容历史API？）：如果`method`字段为`"call"`时，将`params`列表的第一个值作为`method`使用。
```json
{
    "jsonrpc":"2.0",
    "id":0,
    "method":"call",
    "params":[
        "database_api",
        "get_content",
        [
            "htcptc",
            "it-s-a-great-date-8f5a62660a36f"
        ]
    ]
}
```

`steemd`下主要的API插件都位于源码的[`libraries/plugins/apis/`](https://github.com/steemit/steem/tree/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/apis)
目录下。

## API类的声明
下面是一个API类的基本声明：
```cpp
class account_by_key_api
{
   public:
      account_by_key_api();
      ~account_by_key_api();

      DECLARE_API( (get_key_references) )

   private:
      std::unique_ptr< detail::account_by_key_api_impl > my;
};
```

`steemd`自己用宏实现一套API声明注册的框架，每个API，对应一个class，如`account_by_key_api`。这个class并
不实现任何API的逻辑，可以为是这个框架必须的一个容器，每个API下的方法对应的实现，有存储在class内的`my`指
针上，该指针必须使用这个名称。然后需要对外暴露哪些方法，则使用`DECLARE_API`/`DEFINE_READ_APIS`来声明、
实现这个代理层，然后其生成的方法会调用`my`指针指向的实现类里对应的方法。

该类中每个方法通过[`DECLARE_API`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/include/steem/plugins/json_rpc/utility.hpp#L11-L34)
来宏来声明，支持一次声明多个方法。例如:
```cpp
  DECLARE_API( (push_block) (push_transaction) )
```

该宏的展开是：
```
# 对于每个method都声明这么对应一个函数：
${method}_return ${method}(const ${method}_args & arg, bool lock = false );
...

# 然后再添加一个模板
template< typename Lambda >
void for_each_api( Lambda&& callback )
{
  # 对于每个method都声明这么对应的一个代码块
  {
    typedef std::remove_pointer<decltype(this)>::type this_type;

    callback(*this, "${method}", &this_type::${method},
      static_cast< ${method}_args *>(nullptr),
      static_cast< ${method}_return *>(nullptr)
    );
  }
  ...
}
```

拿上面的`account_by_key_api`来实例展开就是：
```cpp
class account_by_key_api
{
   public:
      account_by_key_api();
      ~account_by_key_api();

      get_key_references_return get_key_references(const get_key_references_args & arg, bool lock = false );

      template< typename Lambda >
      void for_each_api( Lambda&& callback )
      {
         {
            typedef std::remove_pointer<decltype(this)>::type this_type;

            callback(*this, "get_key_references", &this_type::get_key_references,
               static_cast< get_key_references_args *>(nullptr),
               static_cast< get_key_references_return *>(nullptr)
            );
         }
      }

   private:
      std::unique_ptr< detail::account_by_key_api_impl > my;
};
```
其中`for_each_api`中的代码块其实还能再进行一次"展开"：
```cpp
class account_by_key_api
{
   public:
      account_by_key_api();
      ~account_by_key_api();

      get_key_references_return get_key_references(const get_key_references_args & arg, bool lock = false );

      template< typename Lambda >
      void for_each_api( Lambda&& callback )
      {
            callback(*this, "get_key_references", &account_by_key_api::get_key_references,
               static_cast< get_key_references_args *>(nullptr),
               static_cast< get_key_references_return *>(nullptr)
            );
      }

   private:
      std::unique_ptr< detail::account_by_key_api_impl > my;
};
```

当然，`get_key_references_args`、`get_key_references_return`这些类还是要靠[手动声明](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/apis/account_by_key_api/include/steem/plugins/account_by_key_api/account_by_key_api.hpp#L17-L25)的。
```cpp
struct get_key_references_args
{
   std::vector< steem::protocol::public_key_type > keys;
};

struct get_key_references_return
{
   std::vector< std::vector< steem::protocol::account_name_type > > accounts;
};

FC_REFLECT( steem::plugins::account_by_key::get_key_references_args, (keys) )
FC_REFLECT( steem::plugins::account_by_key::get_key_references_return, (accounts) )
```
其中`FC_REFLECT`是一个宏，用模板函数特化和TypeTraits来实现静态的反射，具体的实现，可以参考：

然后我们可以使用以下"反射"函数：
```cpp
const char* fc::get_typename<TYPE>::name(); # 返回类型TYPE的名称
                                            # fc::get_typename<steem::plugins::account_by_key::get_key_references_args>返回字符串"steem::plugins::account_by_key::get_key_references_args"
fc::reflector<TYPE>::is_defined::value      # 表示TYPE是否定义过，当TYPE使用该类宏声明过，value为true，否则为false
                                            # fc::reflector<steem::plugins::account_by_key::get_key_references_args>::is_defined::value == true
fc::reflector<TYPE>::is_enum::value         # 表示TYPE是否是enum
                                            # fc::reflector<steem::plugins::account_by_key::get_key_references_args>::is_enum::value == false
fc::reflector<TYPE>::local_member_count     # 是一个enum，其值为当时TYPE使用FC_REFLECT声明时，其后面声明了几个成员，注意并不是TYPE实际拥有的成员数
                                            # fc::reflector<steem::plugins::account_by_key::get_key_references_args>::local_member_count == 1
fc::reflector<TYPE>::total_member_count     # 使用FC_REFLECT声明的TYPE的total_member_count == local_member_count，其他方式声明的不一定
fc::reflector<TYPE>::visit( Visitor )       # 使用一个Visitor()遍历TYPE注册的成员类型
```

## API类的实现
API类方法的实现使用宏[`DEFINE_READ_APIS`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/include/steem/plugins/json_rpc/utility.hpp#L78-L79)
来生成。
```cpp
DEFINE_READ_APIS( account_by_key_api, (get_key_references) )
```
其展开为：
```cpp
${method}_return ${api_class}::${method} (const ${method}_args & args, bool lock)
{
  if (lock)
  {
    return my->_db.with_read_lock( [&args, this](){ return my->${method}(args); } );
  }
  else
  {
    return my->${method}(args);
  }
}
```
拿上面的`account_by_key_api`来实例展开就是：
```cpp
get_key_references_return account_by_key_api::get_key_references (const get_key_references_args & args, bool lock)
{
  if (lock)
  {
    return my->_db.with_read_lock( [&args, this](){ return my->get_key_references(args); } );
  }
  else
  {
    return my->get_key_references(args);
  }
}
```
其中`my->get_key_references`就是实际逻辑的实现函数，有用户自己实现。

## API的注册
API自己实现
