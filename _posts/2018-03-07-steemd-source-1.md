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
用户实现这2个类时，需要满足3个要求：
1. 这2个类必须调用`FC_REFLECT`进行反射
2. 调用`FC_REFLECT`进行反射时，暴露的类成员必须是public的。
3. 使用`FC_REFLECT`反射暴露的每个成员的类型如果不是buildin类型，则也必须使用`FC_REFLECT`进行声明。
这3点一定需要记住，否则在注册时，构造`fc::variant`时，会实例化失败。

其中`FC_REFLECT`是一个宏，用模板函数特化和TypeTraits来实现静态的反射，具体的实现，可以参考：[1](https://github.com/steemit/fc/tree/e3e2912760884c08513626fad5b50339b16da8ef/include/fc/reflect)

然后我们可以使用以下"反射"函数：
```
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
API类方法的实现使用宏[`DEFINE_READ_APIS`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/include/steem/plugins/json_rpc/utility.hpp#L78-L79)/[`DEFINE_WRITE_APIS`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/include/steem/plugins/json_rpc/utility.hpp#L81-L82)/[`DEFINE_LOCKLESS_APIS`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/include/steem/plugins/json_rpc/utility.hpp#L84-L85)
来生成。例如：
```cpp
DEFINE_READ_APIS( account_by_key_api, (get_key_references) )
```
其展开为：
```
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
另外2个宏展开的函数类似，区别在于`my->_db`锁上的处理。

拿上面的`account_by_key_api`来实例展开就是：
```cpp
get_key_references_return account_by_key_api::get_key_references(
		const get_key_references_args & args, bool lock)
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
API类声明、实现好了后，`JSON RPC`模块并不知该API的存在，因此将API注册到`JSON RPC`的模块插件里。该注册
时机为API类实例化时，在API类的构造函数中，通过宏`JSON_RPC_REGISTER_API`将API方法注册到`JSON RPC`插件里。
```cpp
account_by_key_api::account_by_key_api(): my( new detail::account_by_key_api_impl() )
{
   JSON_RPC_REGISTER_API( STEEM_ACCOUNT_BY_KEY_API_PLUGIN_NAME );
}
```

该宏展开后是：
```cpp
account_by_key_api::account_by_key_api(): my( new detail::account_by_key_api_impl() )
{
   steem::plugins::json_rpc::detail::register_api_method_visitor vtor("account_by_key_api");
   for_each_api(vtor);
}
```

其调用了前面宏生成的`for_each_api`函数，在该函数中会调用
`void steem::plugins::json_rpc::detail::register_api_method_visitor::operator()`方法进行注册。其实现为：
```cpp
template< typename Plugin, typename Method, typename Args, typename Ret >
void operator()( Plugin& plugin, const std::string& method_name, Method method, Args* args, Ret* ret )
{
  _json_rpc_plugin.add_api_method( _api_name, method_name,
    [&plugin,method]( const fc::variant& args ) -> fc::variant
    {
      return fc::variant( (plugin.*method)( args.as< Args >(), true ) );
    },
    api_method_signature{ fc::variant( Args() ), fc::variant( Ret() ) }
  );
}
```
其中的`_json_rpc_plugin`是JSON RPC插件的一个引用，通过`appbase::app().get_plugin< steem::plugins::json_rpc::json_rpc_plugin >()`
获得。[`fc::variant`](https://github.com/steemit/fc/blob/e3e2912760884c08513626fad5b50339b16da8ef/include/fc/variant.hpp#L173-L346)
是一个类型容器，可以存储一些任意类型。其主要使用的是枚举方法实现，与`std::variant`和`boost::variant`
完全不同的。

这个注册过程就是一次将上面生成的每个API方法的的名称、输入参数、输出参数传递给`JSON RPC`插件的
`add_api_method`方法。

拿前面的`account_by_key_api`举例就是：
```cpp
  // 该函数的几个输入参数为：
  // API类名称
  // 要注册的方法函数名称
  // 一个lambda表达式，该表达式返回一个存储该API类的该方法返回对象的fc::variant
  // struct api_method_signature 对象，该对象有2个成员，分别是存储要注册方法输入和输出对象的fc::variant
  _json_rpc_plugin.add_api_method("account_by_key_api", "get_key_references"
    [&account_by_key_api插件类的引用, account_by_key_api::get_key_references函数指针]( const fc::variant & args) -> fc::variant
    {
      return fc::variant( account_by_key_api::get_key_references(args.as<get_key_references_args *>(), true) );
    },
    api_method_signature{fc::variant( get_key_references_args() ), fc::variant( get_key_references_return() )}
  );
```

注意：使用非build类型构造`fc::variant`时，该类型必须之前使用`FC_REFLECT`之类的宏构建一系列反射所需的声明。
因为他们在构造`fc::variant`需要使用到[`void fc::to_variant( const T& o, variant& v )`](https://github.com/steemit/fc/blob/e3e2912760884c08513626fad5b50339b16da8ef/include/fc/reflect/variant.hpp#L97-L101)，
里面会调用到反射的一些东西。最终会构造成构造成存储了一个`fc::mutable_variant_object`的`fc::variant`：
```cpp
// mutable_variant_object 中 std::vector< entry > 放置的是先前该类使用FC_REFLECT声明反射时暴露
// 的每个成员名称，和成员类型值
variant::variant( mutable_variant_object obj)
{
   *reinterpret_cast<variant_object**>(this)  = new variant_object(fc::move(obj));
   set_variant_type(this,  object_type );
}
```

接下来我们需要看`JSON RPC`插件的`add_api_method`这个方法的内部逻辑。其最终会调用
`steem::plugins::json_rpc::detail::json_rpc_plugin_impl::add_api_method`这个方法。
`steem::plugins::json_rpc::detail::json_rpc_plugin_impl`类内部有3个成员用来存储`add_api_method`被调用时
传入的参数，即API类名，方法名，方法输入/输出参数签名。
```
  std::map< string, api_description >                          _registered_apis;
  std::vector< string >                                        _methods;
  std::map< string, std::map< string, api_method_signature > > _method_sigs;
```
此时，`JSON RPC`插件就知道了各个API类注册的API类、方法名、参数签名等信息。

## 已注册API的调用
接下来，我们需要了解`JSON RPC`插件是如何利用这些注册信息来进行对应的方法调用。`JSON RPC`插件并不负责
HTTP通讯，HTTP通讯由`webserver`插件负责，这个模块留在以后再进行分析。

当一个HTTP请求到达服务器时，会通过`webserver`插件进行处理，然后调用`JSON RPC`插件的[`string json_rpc_plugin::call( const string& message )`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L426-L466)
方法。

该方法先将获得的数据解析成JSON，存储在`fc::variant`中，然后调用[`json_rpc_response json_rpc_plugin_impl::rpc(const fc::variant& message)`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L327-L381)。
经过一些细节处理，然后调用[`void json_rpc_plugin_impl::rpc_jsonrpc( const fc::variant_object& request, json_rpc_response& response )`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L265-L325)。
该方法校验解析得到的JSON数据是否符合[`JSON-RPC 2.0`](http://wiki.geekdream.com/Specification/json-rpc_2.0.html)
的规范，然后从数据中取出调用参数和方法指针，此处[获得的方法指针](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L200-L209)
就是`_json_rpc_plugin.add_api_method`注册的那个lambda表达式。然后再[调用该lambda表达式](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L292-L293)，
即完成API的调用。

## 获取全部注册的API方法
`JSON RPC`插件对外暴露了2个方法[`get_methods`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L178-L182)、[`get_signature`](https://github.com/steemit/steem/blob/42e2d95ec09d1695ec1b392d47a2e44612815cf0/libraries/plugins/json_rpc/json_rpc_plugin.cpp#L184-L198)
用于让用户查看正在运行的服务器上注册的API方法和每个方法的签名。

我们可以通过以下命令来调用:
```sh
> curl -s http://xx.xx.xx.xx:8090/rpc -d '{"jsonrpc": "2.0", "method": "jsonrpc.get_methods", "params": {}, "id": 11}' | python -mjson.tool
> curl -s http://xx.xx.xx.xx:8090/rpc -d '{"jsonrpc": "2.0", "method": "jsonrpc.get_signature", "params": {"method":"block_api.get_block"}, "id": 11}' |python -mjson.tool
```

## 实现自己的插件和API

