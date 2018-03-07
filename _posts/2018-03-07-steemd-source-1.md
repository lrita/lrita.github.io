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
其中该类中每个方法通过[`DECLARE_API`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/include/steem/plugins/json_rpc/utility.hpp#L11-L34)
来宏来声明，支持一次声明多个方法。例如:
```cpp
      DECLARE_API(
         (push_block)
         (push_transaction) )
```

该宏的展开是：
```
# 对于每个method都声明这么对应一个函数：
${method}_return ${method}(const ${method}_args & arg, bool lock = false );
...

# 然后在添加一个模板
template< typename Lambda >
void for_each_api( Lambda&& callback )
{
   # 对于每个method都声明这么对应的一个代码块
  {
    typedef std::remove_pointer<decltype(this)>::type this_type;

    callback(*this, "${method}", &this_type::method,
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

            callback(*this, "get_key_references", &this_type::method,
               static_cast< get_key_references_args *>(nullptr),
               static_cast< get_key_references_return *>(nullptr)
            );
         }
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

## API类的实现

