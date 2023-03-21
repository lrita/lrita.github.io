---
layout: post
title: 使用clang-tidy在CI中自动修复代码中简单问题和检测代码问题
categories: [c++]
description: c++
keywords: c++ clang-tidy
---

一个几十人同时参与开发的C++项目，可以通过引入`clang-tidy`来帮助团队提升编码规范，可以讲题放到CI任务中，自动修复、检测一些常见问题，可以极大程度解放人力，并且缓解矛盾，~~总有一些人瞎写、记性差，长久训练也无法提升，只靠code review也要在这些人身上消耗大量人力，而且这些人通常还会不停的ping你，企图用上线时间迫使你放松一些，赶紧把他那些不用心的代码合进去。用一个冷冰冰的CI也可以很方便的堵住这些人的嘴~~。

下面是我基于`clang-tidy`的一些实践总结：
- 首先`clang-tidy`基本处于能用、可用的状态，不完美，bug还不少，但是也基本足够了。
- `clang-tidy`能够检查很大一部分低幼问题
- `clang-tidy`拥有自动修复模式(`-fix`)，但bug比较多，经常胡乱修复，下面案例中提供了一些我常用的配置项，这些项能够稳定自动修复。

我们需要用到的工具有：
- `clang-tidy`二进制程序，编译安装[llvm-project](https://github.com/llvm/llvm-project)就可以得到，单线程程序、每次检测一个源码文件；
- `run-clang-tidy.py`脚本，貌似不会出现在llvm-project安装目录里，得从源码中拷贝出来。这个脚本主要并发执行N个`clang-tidy`进程；
- `clang-tidy`的配置(`.ci/.clang-tidy`)、脚手架脚本等

clang-tidy 配置
```
---
# 配置clang-tidy配置检测项，带'-'前缀的为disable对应的检测，否则为开启。这里主要是关闭一些用处不大，或者存在bug、假阳性的检查项
Checks: '*,
    -llvm-*,
    -llvmlibc-*,
    -altera-*,
    -android-*,
    -boost-*,
    -darwin-*,
    -fuchsia-*,
    -linuxkernel-*,
    -objc-*,
    -portability-*,
    -zircon-*,
    -clang-analyzer-osx*,
    -clang-analyzer-optin.cplusplus.UninitializedObject,
    -clang-analyzer-optin.cplusplus.VirtualCall,
    -clang-analyzer-core.NullDereference,
    -clang-analyzer-cplusplus.NewDelete,
    -clang-analyzer-cplusplus.PlacementNew,
    -clang-analyzer-cplusplus.NewDeleteLeaks,
    -clang-analyzer-cplusplus.Move,
    -clang-diagnostic-unused-parameter,
    -cppcoreguidelines-*,
    cppcoreguidelines-explicit-virtual-functions,
    cppcoreguidelines-special-member-functions,
    -cert-err58-cpp,
    -cert-env33-c,
    -cert-dcl37-c,
    -cert-dcl51-cpp,
    -google-runtime-int,
    -google-readability-casting,
    -google-readability-function-size,
    -google-readability-todo,
    -google-readability-braces-around-statements,
    -google-build-using-namespace,
    -readability-magic-numbers,
    -readability-implicit-bool-conversion,
    -readability-function-cognitive-complexity,
    -readability-isolate-declaration,
    -readability-convert-member-functions-to-static,
    -readability-container-size-empty,
    -readability-function-size,
    -readability-qualified-auto,
    -readability-make-member-function-const,
    -readability-named-parameter,
    -modernize-use-trailing-return-type,
    -modernize-avoid-c-arrays,
    -modernize-use-nullptr,
    -modernize-replace-disallow-copy-and-assign-macro,
    -modernize-use-bool-literals,
    -modernize-use-equals-default,
    -modernize-use-default-member-init,
    -modernize-use-auto,
    -modernize-loop-convert,
    -modernize-deprecated-headers,
    -modernize-raw-string-literal,
    -misc-no-recursion,
    -misc-unused-parameters,
    -misc-redundant-expression,
    -misc-non-private-member-variables-in-classes,
    -hicpp-*,
    hicpp-exception-baseclass,
    -performance-no-int-to-ptr,
    -bugprone-easily-swappable-parameters,
    -bugprone-implicit-widening-of-multiplication-result,
    -bugprone-integer-division,
    -bugprone-exception-escape,
    -bugprone-reserved-identifier,
    -bugprone-branch-clone,
    -bugprone-narrowing-conversions,
'
# 将警告转为错误
WarningsAsErrors: '*,-misc-non-private-member-variables-in-classes'
FormatStyle: file
# 过滤检查哪些头文件，clang-tidy会把源码依赖的头文件列出来都检查一遍，所以要屏蔽大量第三方库中的头文件
# 参考 https://stackoverflow.com/questions/71797349/is-it-possible-to-ignore-a-header-with-clang-tidy
# 该正则表达式引擎为llvm::Regex，支持的表达式较少，(?!xx)负向查找等都不支持
HeaderFilterRegex: '(xxx/include)*\.h$'
# 具体一些检查项的配置参数，可以参考的：
# https://github.com/envoyproxy/envoy/blob/main/.clang-tidy
# https://github.com/ClickHouse/ClickHouse/blob/d1d2f2c1a4979d17b7d58f591f56346bc79278f8/.clang-tidy
CheckOptions:
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: readability-identifier-naming.EnumCase
    value: CamelCase
  - key: readability-identifier-naming.LocalVariableCase
    value: lower_case
  - key: readability-identifier-naming.StaticConstantCase
    value: aNy_CasE
  - key: readability-identifier-naming.PrivateMemberCase
    value: lower_case
  - key: readability-identifier-naming.PrivateMemberSuffix
    value: _
  - key: readability-identifier-naming.ProtectedMethodCase
    value: lower_case
  - key: readability-identifier-naming.ProtectedMethodSuffix
    value: _
  - key: readability-braces-around-statements.ShortStatementLines
    value: 2
  - key: readability-uppercase-literal-suffix.NewSuffixes
    value: 'f;u;ul'
  # Ignore GoogleTest function macros.
  - key: readability-identifier-naming.FunctionIgnoredRegexp
    value: '(TEST|TEST_F|TEST_P|INSTANTIATE_TEST_SUITE_P|MOCK_METHOD|TYPED_TEST)'
  - key: performance-move-const-arg.CheckTriviallyCopyableMove
    value: 0
  - key: cppcoreguidelines-special-member-functions.AllowSoleDefaultDtor
    value: 1
  - key: cppcoreguidelines-special-member-functions.AllowMissingMoveFunctions
    value: 1
  - key: cppcoreguidelines-special-member-functions.AllowMissingMoveFunctionsWhenCopyIsDeleted
    value: 1
```

封装脚本：封装脚本将`clang-tidy`的2个模式封装成了2个函数，可以在CI环境中依次调用两个函数，其中`auto_fix_simple_code`是修复模式，`clang_tidy_check_all`是检查模式。
```sh
#!/bin/bash

function say() {
  echo ">> $(date '+%Y-%m-%d %H:%M:%S') $*"
}

function cmd() {
  say "@$*"
  # shellcheck disable=SC2068
  $@ 2>&1
}

function join_by() {
  local IFS="$1"
  shift
  echo "$*"
}

function auto_fix_simple_code() {
  # 可以被自动修复的检查项，下面是一些能够稳定修复的常见错误
  AUTO_FIX_CHECKS_CFG=(
    "-*"
    "modernize-use-nullptr"
    "modernize-use-override"
    "modernize-use-using"
    "modernize-make-shared"
    "boost-use-to-string"
    "readability-container-size-empty"
    "readability-redundant-access-specifiers"
    "readability-redundant-string-cstr"
    "readability-redundant-string-init"
    "readability-redundant-smartptr-get"
    "readability-redundant-control-flow"
    "google-readability-namespace-comments"
    "performance-unnecessary-copy-initialization"
    "performance-for-range-copy"
    "performance-noexcept-move-constructor"
    "clang-analyzer-deadcode.DeadStores"
  )
  AUTO_FIX_CHECKS=$(join_by "," "${AUTO_FIX_CHECKS_CFG[@]}")
  #
  run-clang-tidy.py -p "$BUILD_DIRECTORY" \
    -checks="$AUTO_FIX_CHECKS" \
    -fix $FILE \
    > /tmp/clang-tidy-fix.log 2>&1

  if [[ -n "${GITLAB_CI}" && "$(git status --short | wc -l)" != "0" ]]; then
    set -e +o pipefail
    # 存在被自动修复的变更，提交修复变更代码
    cmd git add -u
    cmd git commit -m "自动修复常规问题"
    cmd git push "http://${CI_USER}:${CI_PRIVATE_TOKEN}@${CI_REPOSITORY_URL#*@}" "HEAD:${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME}"
    exit 0
  fi
}

function clang_tidy_check_all() {
  # 检查仍然存在的问题
  say .ci/run-clang-tidy.py -p="$BUILD_DIRECTORY" \
    -config-file=".ci/.clang-tidy" $FILE
  .ci/run-clang-tidy.py -p="$BUILD_DIRECTORY" \
    -config-file=".ci/.clang-tidy" $FILE \
    > /tmp/clang-tidy-issue.log 2>&1

  if [[ -n "${GITLAB_CI}" ]]; then
    {
      echo "clang-tidy 检测结果："
      echo '```'
      grep -A 2 -E "error:.*\[.*\]" /tmp/clang-tidy-issue.log
      echo '```'
      echo "详情请点击pipeline⭕️图标进行查看"
    } > /tmp/clang-tidy-summary.log
    if [[ $(wc -l < "/tmp/clang-tidy-summary.log") -gt 4 ]]; then
      cmd add_comment "@/tmp/clang-tidy-summary.log" # add_comment 是CI中提供的一个命令，给对应MR中添加评论
      exit 255                                       # 使CI任务失败
    fi
  else
    {
      echo "clang-tidy 检测结果："
      echo '```'
      grep -E "error:.*\[.*\]" /tmp/clang-tidy-issue.log | grep -Eo "\[.*\]" | sort | uniq -c | sort -n
      echo '```'
      echo "详情请点击pipeline⭕️图标进行查看"
    } > /tmp/clang-tidy-summary.log
    cat /tmp/clang-tidy-issue.log
  fi
}

BUILD_DIRECTORY="./build"                  # cmake执行目录
SOURCE_DIRECTORY=${CI_PROJECT_DIR:-$(pwd)} # 源码目录
say "build cmake in $BUILD_DIRECTORY ..."
mkdir -p $BUILD_DIRECTORY
# 执行cmake build，-DCMAKE_EXPORT_COMPILE_COMMANDS=ON 使cmake生成单文件编译依赖配置文件，后续clang-tidy执行需要依赖该配置
# 会在cmake build目录下生成一个 compile_commands.jso n文件
cmd cd "$BUILD_DIRECTORY" \
  && cmd cmake "$SOURCE_DIRECTORY" -DCMAKE_BUILD_TYPE:STRING=RelWithDebInfo -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cmd cd "$SOURCE_DIRECTORY"

if [[ -n "${GITLAB_CI}" ]]; then # gitlab CI中会定义GITLAB_CI变量
  # 运行在CI中
  M_SHA1=$(git rev-parse origin/master)
  # 过滤出本次MR中涉及修改的文件
  FILE=$(git diff --name-status "$M_SHA1" | grep -E "^(M|A)\s+(include|src)/.*\.(cc|cpp|h|hpp)$" | awk '!/tests/ { print $2 }')
  [[ "$FILE" == "" ]] && exit 0
else
  # 手动执行
  FILE='.*\.(?:h|cc)*'
fi

case "$1" in
fix)
  auto_fix_simple_code
  ;;
check)
  clang_tidy_check_all
  ;;
esac
```