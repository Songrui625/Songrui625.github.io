---
external: false

title: 简单水水CPU的频率、电源管理

description: introduction to cpu frequency and power management

date: 2023-12-18
---

在云场景下，经常会发现CPU利用率到达100%，但是性能上不去的问题，本质上是频率和功耗上不去。这里从一个程序员，而不是CPU开发人员的视角，简单聊聊CPU的电源管理和频率。这里笔者接触更多的是Intel的CPU，本文也只关注在Intel的CPU上。

# CPU管理电源
作为购买云服务器的客户，肯定希望自己的CPU能充分利用起来，防止性能受限，浪费金钱，而对于一些PC个人用户来说，则希望自己的CPU在电脑待机时风扇不至于狂转不止，月底电费缠身。CPU开发者自然也是想到了这一点，这就是CPU电源管理需要处理的事情的。

CPU广义上存在两个电源模式，一种是性能模式(performance mode)，一种是节能/省电模式(powersave mode)。
## P-states/性能模式(Performance Mode)
性能模式是高级配置和电源接口规范(Advanced Configuration and Power Interface (ACPI))所定义的处理器处于运行状态, 一般称为P-state，以Px表示，x从0开始，P0则是最大功耗和频率，x越大，则频率和功耗越低。

### Intel-pstate
在Linux系统中，暴露给程序员的CPU性能模式管理接口的是intel_pstate子系统，可以在sysfs中的`/sys/devices/system/cpu/intel_pstate`目录查看暴露的接口。
```bash
$ ls /sys/devices/system/cpu/intel_pstate
hwp_dynamic_boost  max_perf_pct  min_perf_pct  no_turbo  num_pstates  status  turbo_pct
```

我们这里主要关注`no_turbo`这个文件，这个是Intel的Turbo Boost技术，我也没想到应该怎么翻译，姑且叫他睿频加速好了，这个文件表示限制CPU的频率是否处于睿频之下，值可以选0和1，默认值为0，即默认开启turbo boost模式，他会在CPU需要时以更快的频率工作。在一些支持SIMD指令集的CPU上，尤其是AVX2、AVX512，运行这些重指令会产生大量的能耗导致CPU降频，这个问题在hacker news上有广泛的[讨论](https://news.ycombinator.com/item?id=24215022)，这是动态频率缩放(dynamic frequency scaling)导致的。

### cpufreq scaling subsystem
我们需要关注的另一个CPU电源管理子系统是cpufreq子系统，他也叫频率伸缩子系统，可以在sysfs的`/sys/devices/system/cpu/cpufreq/policy<x>`目录下查看暴露的接口，当然也可以在`/sys/devices/system/cpu/cpu<0>/cpufreq`目录下找到，后者是前者的符号连接，注意这是针对每一个核心的控制。
```bash
$ ls /sys/devices/system/cpu/cpufreq/policy0/
affected_cpus   cpuinfo_max_freq  cpuinfo_transition_latency                energy_performance_preference  scaling_available_governors  scaling_driver    scaling_max_freq  scaling_setspeed
base_frequency  cpuinfo_min_freq  energy_performance_available_preferences  related_cpus                   scaling_cur_freq             scaling_governor  scaling_min_freq
```

#### 伸缩调节器(scaling governor)
cpufreq subsystem为我们提供了两种伸缩调节器来控制CPU的策略，可以通过`scaling_governor`文件来设置，一种是"powersave"，另一种是"performance"。

当调节器为"powersave"时，cpu的频率会受限于`scaling_min_freq`文件的值，即每次运行的频率不会超过scaling_min_freq。

当调节器为"performance"时，cpu频率则会受限于`scaling_max_freq`文件的值，我们一般会将调节器设置为"performace"模式，防止cpu在运行生产业务处理过慢，调节器处于"powersave"时在同等指令下可能会花费更多的运行时间，造成利用率偏高。

#### 查看CPU频率(frequency scaling)
我们可以通过`scaling_cur_freq`文件查看当前CPU的频率，单位是kHz，这个是在软件层面较为准确地读取CPU频率的途径，他的实现是通过读取MSR寄存器的aperf和mperf的值进行相应计算完成的，具体可以看这个[patch](https://lore.kernel.org/lkml/52f711be59539723358bea1aa3c368910a68b46d.1459485198.git.len.brown@intel.com/)。

除了`scaling_cur_freq`文件，我们还可以通过turbostat、pcm等工具查看CPU的频率。


## C-states/节能模式(Powersave Mode)
聊完CPU的性能模式P-states, 我们来聊一下节能模式，也被称为C-states，有些人还称为空闲(idle)模式，当然这个理解是不对的，C-state以Cn表示，n从0开始，n越大，则CPU的功耗就越低，C6被认为是一个终极节能状态，但有个特例是C0，C0是CPU运行模式，CPU的C-states为C0则标志着CPU核处于活跃状态，在不同代号的CPU上，可能C1也会被认为CPU核处于活跃状态。

我们可以通过sysfs的`/sys/system/device/cpu/cpu<x>/cpuidle`来查看当前系统支持的所有C-states。
```bash
$ ls /sys/devices/system/cpu/cpu0/cpuidle
state0  state1  state2  state3
```
这里的state<x>，x代表并不一定是Cx，可以查看state{x}/name查看该C-state的名称:
```bash
$ cat */name
POLL
C1
C1E
C6
```

我们还可以通过cpupower方便地查看每个C-state的开启和关闭状态，而不是去文件里面一个一个看。
```bash
$ cpupower idle-info
CPUidle driver: intel_idle
CPUidle governor: menu
analyzing CPU 0:

Number of idle states: 4
Available idle states: POLL C1 C1E C6
POLL:
Flags/Description: CPUIDLE CORE POLL IDLE
Latency: 0
Usage: 540710
Duration: 15742526
C1:
Flags/Description: MWAIT 0x00
Latency: 1
Usage: 1883513
Duration: 269397652
C1E:
Flags/Description: MWAIT 0x01
Latency: 2
Usage: 502712325
Duration: 2523210382346
C6 (DISABLED) :
Flags/Description: MWAIT 0x20
Latency: 190
Usage: 0
Duration: 0
```
值得一提的是，某些云厂商会把C6给关闭，这是因为开启C6，可能会导致售卖出去的虚拟机存在算力不一致的现象，一块CPU的最大调度功耗是一定的，当某些CPU核心进入C6 state时，而另外一些CPU核心在运行重负载，例如avx512指令，典型的就是linpack这种应用，那么CPU会集中把功耗调度到运行重负载的CPU核上，性能能上一个档次，当然这是牺牲了其他CPU核心的算力换来的。为了保证售卖出去的虚拟机的CPU核心算力能稳定，云厂商会在Host侧关闭C6，虚拟机内部的也会通过启动参数`intel_idle.max_cstate=1`和`processor.max_cstate=1`来限制CPU核心的C-states最多进入C1 state，而不会进入C6 state。

## 总结
简单水了水CPU电源管理、频率的相关内容，当然还有一些其他例如热设计功耗TDP之类的内容，这些都可以在参考资料里面找到，笔者也是初学者，难免有疏漏之处，欢迎大家指正，`祝大家新年快乐！

# Reference
<https://en.wikipedia.org/wiki/Dynamic_frequency_scaling>   
<https://vstinner.github.io/intel-cpus.html>    
<https://vstinner.github.io/intel-cpus-part2.html>  
<https://www.kernel.org/doc/html/v4.12/admin-guide/pm/cpufreq.html> 
<https://www.kernel.org/doc/html/v4.12/admin-guide/pm/intel_pstate.html>    
<https://www.kernel.org/doc/Documentation/cpu-freq/intel-pstate.txt>    
<https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/c-state.html>    
<https://networkbuilders.intel.com/docs/networkbuilders/intel-turbo-boost-technology-configure-per-core-turbo-overview-technology-guide-1673528561.pdf>     
<https://networkbuilders.intel.com/docs/networkbuilders/power-management-technology-overview-technology-guide-1677587938.pdf>   
<http://gauss.ececs.uc.edu/Courses/c4029/lectures/power_manage.pdf>