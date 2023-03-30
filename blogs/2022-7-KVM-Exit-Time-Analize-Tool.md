# kvm陷出开销分析工具kvmexittime

安全容器和普通容器的区别是安全容器的应用运行在虚拟机里面。虚拟机频繁的exit操作会造成我们所说的虚拟化开销。那么如何量化这些开销呢？在BCC工具集里有一个[kvmexit工具](https://github.com/iovisor/bcc/blob/master/tools/kvmexit_example.txt)，该工具能统计KVM陷出的原因和次数，对于分析kvm陷出的频繁程度有参考价值。而本文要介绍的是能够精细地刻画kvm陷出时间的工具[kvmexittime](https://gitee.com/anolis/sysak/tree/opensource_branch/source/tools/detect/virt/kvmexittime)。

## KVM背景

![logo1](/asserts/img/blog-2022-7-KVMEXITTIME/kvm-exit-process.png)

## kvmexittime实现方法
下图是KVM陷出的过程简图。虚拟机执行KVM Exit从Guest状态退出之后，它可能执行一些I/O处理操作又回到Guest，也可能调度出切换成其他进程。我们需要统计的是执行I/O操作的这段开销，即虚拟机除Guest之外的开销。这样，我们得到一个计算开销的公式：
Overhead = Time(Exit-Entry)-Time(Switch out-Switch in)

![logo2](/asserts/img/blog-2022-7-KVMEXITTIME/kvm-exit-time.png)

为避开BCC方式带来的依赖，kvmtime是基于libbpf实现，目前集成在龙蜥开源社区的开源项目SysAK中。kvmexittime支持对整机、单进程、线程的开销采集。kvmexittime是CO-RE的，进程和线程的ID通过array类型的map传递给相应BPF程序。

## kvmexittime使用

```
$./sysak kvmexittime --help
Usage: kvmexittime [OPTION...]
Trace kvm exit time.

USAGE: kvmexittime [--help] [-p PID] [-t TID] [interval]

EXAMPLES:
    kvmexittime        # trace whole system kvm exit time
    kvmexittime 5      # trace 5 seconds summaries
    kvmexittime -p 123 # trace kvm exit time for pid 123
    kvmexittime -t 123 # trace kvm exit time for tid 123 (use for threads
only)

  -p, --pid=PID              Process PID to trace
  -t, --tid=TID              Thread ID to trace
  -v, --verbose              Verbose debug output
  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version

Mandatory or optional arguments to long options are also mandatory or optional
for any corresponding short options.
```

## 使用案例

```
$./sysak kvmexittime -p 293573
Tracing kvm exit...  Hit Ctrl-C to end.

Exit reason       	Count           	E2E-time        	Sched-out-time  	On-cpu-time
RDMSR             	2               	2767            	0               	2767
HLT               	6               	1553377169      	1553309529      	67640
WRMSR             	28              	17585           	0               	17585
VMCALL            	4               	9157            	0               	9157
Total             	40              	1553406678      	1553309529      	97149
```
