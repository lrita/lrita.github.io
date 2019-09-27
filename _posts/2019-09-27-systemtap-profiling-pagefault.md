---
layout: post
title: 使用perf/SystemTap分析pagefault
categories: [linux, tool, SystemTap, perf]
description: linux SystemTap pagefault perf
keywords: linux SystemTap pagefault perf
---

`pagefault`在使用大量内存的场景下是一个不可忽视的性能损耗，而且在用户态中，该行为是透明的，不好分析和测量，因此必须借助外部工具才能分析。

# perf

我们可以使用`perf`，很轻松的分析出，哪些代码会经常性的触发`pagefault`，以及比重。

首先，我们可以使用以下命令采集`pagefault`发生的次数。

```sh
# -I 1000 每1000ms输出一次，
# -a 采集全部CPU上的事件
# -p <pid> 可以指定进程
> perf stat -e page-faults -I 1000 -a
> perf stat -e page-faults -I 1000 -a -p 10102
```

或者，我们还可以使用[FlameGraph](https://github.com/brendangregg/FlameGraph)更加直观的看到各部分代码触发`pagefault`的比例：

```sh
# 采集进程10102的30秒pagefault触发数据
> perf record -e page-faults -a -p 10102 -g -- sleep 30
# 导出原始数据，此步必须在采集机器上进行，因为需要解析符号。
> perf script > out.stacks
# 下面的步骤就可以移动到其他机器上了
> ./FlameGraph/stackcollapse-perf.pl < out.stacks | ./FlameGraph/flamegraph.pl --color=mem \
    --title="Page Fault Flame Graph" --countname="pages" > out.svg
```

我们使用浏览器来打开`out.svg`就可以直观观察了。

# SystemTap

我们可以使用以下脚本，每 10 秒输出一次相关进程触发的全部`pagefault`异常的类型与耗时：

```c
#!/usr/bin/stap

/**
 * Tested on Linux 3.10 (CentOS 7)
 */

global fault_entry_time, fault_latency_all, fault_latency_type

function vm_fault_str(fault_type: long) {
    if(vm_fault_contains(fault_type, VM_FAULT_OOM))
        return "OOM";
    else if(vm_fault_contains(fault_type, VM_FAULT_SIGBUS))
        return "SIGBUS";
    else if(vm_fault_contains(fault_type, VM_FAULT_MINOR))
        return "MINOR";
    else if(vm_fault_contains(fault_type, VM_FAULT_MAJOR))
        return "MAJOR";
    else if(vm_fault_contains(fault_type, VM_FAULT_NOPAGE))
        return "NOPAGE";
    else if(vm_fault_contains(fault_type, VM_FAULT_LOCKED))
        return "LOCKED";
    else if(vm_fault_contains(fault_type, VM_FAULT_ERROR))
        return "ERROR";
    return "???";
}

probe vm.pagefault {
	if (pid() == target()) {
		fault_entry_time[tid()] = gettimeofday_us()
	}
}

probe vm.pagefault.return {
	if (!(tid() in fault_entry_time)) next
	latency = gettimeofday_us() - fault_entry_time[tid()]
	fault_latency_all <<< latency
	fault_latency_type[vm_fault_str(fault_type)] <<< latency
}

probe timer.s(10) {
	print("All:\n")
	print(@hist_log(fault_latency_all))
	delete(fault_latency_all)

	foreach (type in fault_latency_type+) {
		print(type,":\n")
                print(@hist_log(fault_latency_type[type]))
        }
        delete(fault_latency_type)
}
```
