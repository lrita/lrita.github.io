---
layout: post
title: 字符串解析通用方法
categories: [c++]
description: c++
keywords: c++ utility string
---

最近业务上频繁需要处理多种分隔符拼接成的各种数据，总结了一种抽象程度高、解释度高、使用简洁的通用解析方式。

```cpp
struct Student {
  std::string name;
  float       score1;
  float       score2;
};

std::string f = "Tom@20@0.5@0.6|John@21@0.6@0.6|Jean@16@0.2@0.1";
std::map<std::string, Student> student_map;
// 简洁: 体现在类似的两层数据表达在调用解析的时候，只需要这么简单。
// 抽象程度高: 体现在解析规则的lambda表达式可以支持多种builtin类型的自由组合，用户可以自己根据字符串格式要求自行声明对应格式的lambda表达式
//           然后该会将对应字符串转换成要求的类型，然后触发lambda调用。
// 解释度高: 体现在通过lambda表达式的声明就表达的对应字符串的格式，可以不再进行额外的注释，例如下面的例子，每个字段的含义可以通过lambda表达式
//          的函数声明体现。
SplitString2ValueVec(f, '|', '@', [&](std::string name, int age, float score1, float score2) {
  if (age > 18) {
    student_map.insert(name, Student{name, score1, score2});
  }
})
```

```cpp
#pragma once
#include <tuple>
#include <string>
#include <vector>
#include <string.h>
#include <boost/callable_traits.hpp>
#include <boost/type_traits.hpp>
#include <boost/utility/string_view.hpp>
#include <folly/Traits.h>
#include <folly/functional/ApplyTuple.h>
#include <absl/utility/utility.h>
#include <absl/strings/str_split.h>
#include <absl/strings/numbers.h>

namespace common {
namespace common_detail {
FOLLY_CREATE_HAS_MEMBER_TYPE_TRAITS(has_key_type_traits, key_type);

template <typename T, typename = void>
struct Conv2Type;

template <typename T>
struct Conv2Type<T, std::enable_if_t<std::is_convertible<T, absl::string_view>::value>> {
  bool operator()(absl::string_view m, T *v) {
    *v = T(m);
    return true;
  }
};

template <typename T>
struct Conv2Type<T, std::enable_if_t<std::is_integral<T>::value>> {
  bool operator()(absl::string_view m, T *v) { return absl::SimpleAtoi(m, v); }
};

template <>
struct Conv2Type<double> {
  bool operator()(absl::string_view m, double *v) { return absl::SimpleAtod(m, v); }
};

template <>
struct Conv2Type<float> {
  bool operator()(absl::string_view m, float *v) { return absl::SimpleAtof(m, v); }
};

template <>
struct Conv2Type<bool> {
  bool operator()(absl::string_view m, bool *v) { return absl::SimpleAtob(m, v); }
};

template <size_t N, typename Tuple, typename It, bool IsEnd = (N == std::tuple_size<Tuple>::value)>
struct TupleAssign {
  static bool assign(It it, It end, Tuple &v) {
    static_assert(N < std::tuple_size<Tuple>::value, "bad assign index");
    if (it == end) {
      return false;
    }
    auto vv = *it;
    return TupleAssign<N + 1, Tuple, It>::assign(++it, end, v) &&
           Conv2Type<typename std::tuple_element<N, Tuple>::type>()(vv, &std::get<N>(v));
  }
};

template <size_t N, typename Tuple, typename It>
struct TupleAssign<N, Tuple, It, true> {
  static bool assign(It, It, Tuple &) { return true; }
};

template <typename Callback>
using SplitString2ValueVecCallbackReturnType =
    std::enable_if<std::is_same<void, boost::callable_traits::return_type_t<Callback>>::value ||
                   std::is_same<bool, boost::callable_traits::return_type_t<Callback>>::value>;

template <typename T>
struct RemoveTupleElementReference;

template <typename... T>
struct RemoveTupleElementReference<std::tuple<T...>> {
  using type = std::tuple<boost::remove_cv_ref_t<T>...>;
};

template <typename Callback>
struct SplitString2ValueVecCallbackArgsType {
  using type = typename RemoveTupleElementReference<boost::callable_traits::args_t<Callback>>::type;
};

template <typename T>
struct IsVec : std::false_type { };

template <typename T>
struct IsVec<std::vector<T>> : std::true_type { };

template <typename Callback>
using IsCallbackArgsOnly1Vec = folly::Conjunction<
    folly::bool_constant<std::tuple_size<boost::callable_traits::args_t<Callback>>::value == 1>,
    IsVec<boost::remove_cv_ref_t<
        typename std::tuple_element<0, boost::callable_traits::args_t<Callback>>::type>>>;

struct TupleHasReferenceImpl {
  template <typename... T, typename ReturnType = folly::Disjunction<folly::Conjunction<
                               std::is_lvalue_reference<T>,
                               folly::Negation<std::is_const<std::remove_reference_t<T>>>>...>>
  static ReturnType apply(std::tuple<T...> &&) {
    return {};
  }
};

template <typename Tuple>
using TupleHasReference = decltype(TupleHasReferenceImpl::apply(std::declval<Tuple>()));

template <typename Callback,
          bool HasReference = TupleHasReference<boost::callable_traits::args_t<Callback>>::value>
struct ApplyInvoke {
 public:
  constexpr decltype(auto)
      operator()(Callback &&                                                     cb,
                 typename SplitString2ValueVecCallbackArgsType<Callback>::type &&tuple) const {
    return folly::apply(std::forward<Callback>(cb), std::move(tuple));
  }
};

template <typename Callback>
struct ApplyInvoke<Callback, true> {
 public:
  constexpr decltype(auto)
      operator()(Callback &&                                                     cb,
                 typename SplitString2ValueVecCallbackArgsType<Callback>::type &&tuple) const {
    auto tuple2 = folly::forward_tuple(tuple);
    return folly::apply(std::forward<Callback>(cb), std::move(tuple2));
  }
};

template <typename Callback, typename STRING, typename IsSupport = void>
struct SplitString2ValueVec;

template <typename Callback, typename STRING>
struct SplitString2ValueVec<
    Callback, STRING,
    typename std::enable_if<
        !IsCallbackArgsOnly1Vec<Callback>::value &&
        std::is_same<void, boost::callable_traits::return_type_t<Callback>>::value>::type> {
  // 返回值为void类型的回调函数时:
  void operator()(const STRING &v, char key_value_pair_delimiter, char key_value_delimiter,
                  Callback &&cb) {
    for (const auto &kv_token : absl::StrSplit(v, key_value_pair_delimiter, absl::SkipEmpty())) {
      typename SplitString2ValueVecCallbackArgsType<Callback>::type tuple;
      auto sp = absl::StrSplit(kv_token, key_value_delimiter);
      if (TupleAssign<0, decltype(tuple), typename decltype(sp)::const_iterator>::assign(
              sp.begin(), sp.end(), tuple)) {
        ApplyInvoke<Callback> {}(std::forward<Callback>(cb), std::move(tuple));
      }
    }
  }
};

template <typename Callback, typename STRING>
struct SplitString2ValueVec<
    Callback, STRING,
    typename std::enable_if<
        !IsCallbackArgsOnly1Vec<Callback>::value &&
        std::is_same<bool, boost::callable_traits::return_type_t<Callback>>::value>::type> {
  // 返回值为bool类型的回调函数时，返回值为false时，中断解析。
  void operator()(const STRING &v, char key_value_pair_delimiter, char key_value_delimiter,
                  Callback &&cb) {
    for (const auto &kv_token : absl::StrSplit(v, key_value_pair_delimiter, absl::SkipEmpty())) {
      typename SplitString2ValueVecCallbackArgsType<Callback>::type tuple;
      auto sp = absl::StrSplit(kv_token, key_value_delimiter);
      if (TupleAssign<0, decltype(tuple), typename decltype(sp)::const_iterator>::assign(
              sp.begin(), sp.end(), tuple)) {
        if (!ApplyInvoke<Callback> {}(std::forward<Callback>(cb), std::move(tuple))) {
          break;
        }
      }
    }
  }
};

template <typename Callback, typename STRING>
struct SplitString2ValueVec<
    Callback, STRING,
    typename std::enable_if<
        IsCallbackArgsOnly1Vec<Callback>::value && // 参数为std::vector<T>时
        std::is_same<void, boost::callable_traits::return_type_t<Callback>>::value>::type> {
  // 返回值为void类型的回调函数时:
  void operator()(const STRING &v, char key_value_pair_delimiter, char key_value_delimiter,
                  Callback &&cb) {
    for (const auto &kv_token : absl::StrSplit(v, key_value_pair_delimiter, absl::SkipEmpty())) {
      typename SplitString2ValueVecCallbackArgsType<Callback>::type tuple;
      using element_type = typename std::tuple_element<0, decltype(tuple)>::type;
      bool ok            = true;
      for (const auto &item : absl::StrSplit(kv_token, key_value_delimiter)) {
        typename element_type::value_type v;
        if (!Conv2Type<decltype(v)> {}(item, &v)) {
          ok = false;
          break;
        }
        std::get<0>(tuple).push_back(std::move(v));
      }
      if (ok) {
        ApplyInvoke<Callback> {}(std::forward<Callback>(cb), std::move(tuple));
      }
    }
  }
};

template <typename Callback, typename STRING>
struct SplitString2ValueVec<
    Callback, STRING,
    typename std::enable_if<
        IsCallbackArgsOnly1Vec<Callback>::value && // 参数为std::vector<T>时
        std::is_same<bool, boost::callable_traits::return_type_t<Callback>>::value>::type> {
  // 返回值为bool类型的回调函数时，返回值为false时，中断解析。
  void operator()(const STRING &v, char key_value_pair_delimiter, char key_value_delimiter,
                  Callback &&cb) {
    for (const auto &kv_token : absl::StrSplit(v, key_value_pair_delimiter, absl::SkipEmpty())) {
      typename SplitString2ValueVecCallbackArgsType<Callback>::type tuple;
      using element_type = typename std::tuple_element<0, decltype(tuple)>::type;
      bool ok            = true;
      for (const auto &item : absl::StrSplit(kv_token, key_value_delimiter)) {
        typename element_type::value_type v;
        if (!Conv2Type<decltype(v)> {}(item, &v)) {
          ok = false;
          break;
        }
        std::get<0>(tuple).push_back(std::move(v));
      }
      if (ok) {
        if (!ApplyInvoke<Callback> {}(std::forward<Callback>(cb), std::move(tuple))) {
          break;
        }
      }
    }
  }
};

template <typename Callback, typename StringType>
inline void SplitString2ValueVecF(const StringType &v, char vec_vec_delimiter, char value_delimiter,
                                  Callback &&cb) {
  SplitString2ValueVec<Callback, StringType,
                       typename SplitString2ValueVecCallbackReturnType<Callback>::type> {}(
      v, vec_vec_delimiter, value_delimiter, std::forward<Callback>(cb));
}

template <typename Callback>
inline void SplitString2ValueVecF(const butil::StringPiece &v, char vec_vec_delimiter,
                                  char value_delimiter, Callback &&cb) {
  SplitString2ValueVecF<Callback, absl::string_view>(absl::string_view {v.data(), v.length()},
                                                     vec_vec_delimiter, value_delimiter,
                                                     std::forward<Callback>(cb));
}

template <typename Callback>
inline void SplitString2ValueVecF(const boost::string_view &v, char vec_vec_delimiter,
                                  char value_delimiter, Callback &&cb) {
  SplitString2ValueVecF<Callback, absl::string_view>(absl::string_view {v.data(), v.length()},
                                                     vec_vec_delimiter, value_delimiter,
                                                     std::forward<Callback>(cb));
}

template <typename MAP, typename STRING>
struct SplitString2KeyValueMap {
  void operator()(const STRING &v, char key_value_pair_delimiter, char key_value_delimiter,
                  MAP &map) {
    static_assert(common_detail::has_key_type_traits<MAP>::value,
                  "map type must have key_type alias.");
    static_assert(absl::strings_internal::HasMappedType<MAP>::value,
                  "map type must have mapped_type alias.");
    for (const auto &kv_token : absl::StrSplit(v, key_value_pair_delimiter, absl::SkipEmpty())) {
      std::pair<absl::string_view, absl::string_view> kv =
          absl::StrSplit(kv_token, absl::MaxSplits(key_value_delimiter, 1));
      typename MAP::key_type    key = {};
      typename MAP::mapped_type val = {};
      if (common_detail::Conv2Type<decltype(key)>()(kv.first, &key) &&
          common_detail::Conv2Type<decltype(val)>()(kv.second, &val)) {
        map.emplace(std::move(key), std::move(val));
      }
    }
  }
};

template <typename MAP>
struct SplitString2KeyValueMap<MAP, butil::StringPiece> {
  void operator()(const butil::StringPiece &v, char key_value_pair_delimiter,
                  char key_value_delimiter, MAP &map) {
    SplitString2KeyValueMap<MAP, absl::string_view>()(
        absl::string_view {v.data(), v.size()}, key_value_pair_delimiter, key_value_delimiter, map);
  }
};
} // namespace common_detail

// SplitString2ValueVec 可以将字符串按照回调lambda表达的的输入参数的类型，按需分段解析成对应的
// 类型，然后调用对应的lambda表达式。可以极大地简化字符串解析。
//
// vec_vec_delimiter 为外层字符串格式的分隔符
// value_delimiter 为内层字符串格式的分隔符
//
// 例如：当 vec_vec_delimiter='@',
// value_delimiter=','，被解析字符串为"Tom,1,0.5@Jame,2,0.8@Bean,3,1.0"， 回调函数为 :
// [](std::string name, int index, double score) {
//    std::cout << "name: " << name << " index:" << index << " score:" << score << std::endl;
// }
// 时，则会将字符串解析为按外层分隔符'@'解析为3段，然后再按内层分隔符','将每一段解析成3个三个段，
// 然后分别转换为对应类型std::string int double，然后调用对应的回调函数。然后回调函数分别输出:
// name:Tom index:1 score:0.5
// name:Jame index:2 score:0.8
// name:Bean index:3 score:1.0
//
// 当第二层字符串为同一类型T且个数不定时，传入的lambda可以将接受的参数指定为对应类型的std::vector<T>，
// 这样该函数会将第二层字符串解析为对应类型并传入lambda。
//
// 例如:
// std::string xxx = "1.0@1.1@1.2|2.1@2.2@2.3@2.4";
// common::SplitString2ValueVec(xxx, '|', '@', [](std::vector<double> vec) {
//   std::cout << "vec size=" << vec.size();
//   for (const auto & v : vec) {
//      std::cout << " v=" << v << ",";
//   };
//   std::cout << std::endl;
// });
// 则会输出：
// vec size=3 v=1.0 v=1.1 v=1.2
// vec size=4 v=2.1 v=2.2 v=2.3 v=2.4
template <typename Callback, typename StringType>
inline void SplitString2ValueVec(const StringType &v, char vec_vec_delimiter, char value_delimiter,
                                 Callback &&cb) {
  common_detail::SplitString2ValueVecF(v, vec_vec_delimiter, value_delimiter,
                                       std::forward<Callback>(cb));
}
// 例如：当 key_value_pair_delimiter='@', key_value_delimiter='='，map=std::map<std::string,
// int>时， 将 "abc=1@efg=2@xyz=3" 拆解成 map = { {"abc", 1}, {"efg", 2}, {xyz, 3} }
// 支持任意map类型，比如:
//      std::map<std::string, std::string>
//      std::map<int, int>
//      std::unordered_map<int, int>
// 等
template <typename MAP, typename STRING>
inline void SplitString2KeyValueMap(const STRING &v, char key_value_pair_delimiter,
                                    char key_value_delimiter, MAP &map) {
  common_detail::SplitString2KeyValueMap<MAP, STRING>()(v, key_value_pair_delimiter,
                                                        key_value_delimiter, map);
}
} // namespace common

```
