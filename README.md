md

# PouchContainer supporting LXCFS to achieve high-reliability container isolation

## Introduction
PouchContainer is an open-source runtime container software developed by Alibaba. The latest released version is 0.3.0, located at [https://github.com/alibaba/pouch](https://github.com/alibaba/pouch). PouchContainer is designed to support LXCFS to realize highly reliable container separation. While Linux adopted cgroup technology to realize resource separation, the problem that users obtain host information when trying to read files in /proc/meminfo/, caused by host machine's file system still mounting in container, remains unresolved. The lack of `/proc view isolation` will cause a series of problems such as stalling or obstructing enterprise business containerization. LXCFS ([https://github.com/lxc/lxcfs](https://github.com/lxc/lxcfs)) is an open-source file system solution for resolving `/proc view isolation` issue, making the container acting like a traditional virtual machine in presentation layer. This article will first introduce the appropriate business scene for LXCFS and then introduce how LXCFS works in PouchContainer. 


## LXCFS Business Scene
In the age of physical machine and virtual machine, the Alibaba developed a internal toolbox including compiler packing, application deployment, unified monitoring. These tools have been providing stable services for applications that deploy physical machines and virtual machines. 


### Monitoring and Operational tools
Most monitoring tools rely on the /proc file system to retrieve system information. In the example of Alibaba, part of the infrastructural monitoring tools collect informations through tsar（[https://github.com/alibaba/tsar](https://github.com/alibaba/tsar)). However, collecting memory and CPU information in tsar depends on /proc file system. We can download tsar source code to learn how tsar uses files under /proc. 

```
$ git remote -v
origin https://github.com/alibaba/tsar.git (fetch)
origin https://github.com/alibaba/tsar.git (push)
$ grep -r cpuinfo .
./modules/mod_cpu.c:    if ((ncpufp = fopen("/proc/cpuinfo", "r")) == NULL) {
:tsar letty$ grep -r meminfo .
./include/define.h:#define MEMINFO "/proc/meminfo"
./include/public.h:#define MEMINFO "/proc/meminfo"
./info.md:内存的计数器在/proc/meminfo,里面有一些关键项
./modules/mod_proc.c:    /* read total mem from /proc/meminfo */
./modules/mod_proc.c:    fp = fopen("/proc/meminfo", "r");
./modules/mod_swap.c: * Read swapping statistics from /proc/vmstat & /proc/meminfo.
./modules/mod_swap.c:    /* read /proc/meminfo */
$ grep -r diskstats .
./include/public.h:#define DISKSTATS "/proc/diskstats"
./info.md:IO的计数器文件是:/proc/diskstats,比如:
./modules/mod_io.c:#define IO_FILE "/proc/diskstats"
./modules/mod_io.c:FILE *iofp;                     /* /proc/diskstats*/
./modules/mod_io.c:    handle_error("Can't open /proc/diskstats", !iofp);
```

It is obvious that tsar's monitoring of process, IO, and cpu relies on /proc file system. 

When the information provided by /proc file system is from host machine, these monitorings cannot monitor the information in container. To satisfy business demand to appropriate container monitoring, it is even nessary to develop another set of monitoring tools specifically for a container. This issue will, in nature, stall or even obstruct the containerization of enterprise existing business. Therefore, container technology must have the compatibility of original existing monitoring tools to avoid develop new tools everytime and preserve nice user interface that customs to Engineers' user habits. 

PouchContainer is a tool that support LXCFS and get rid of issues listed above. PouchContainer resolves listed issues by transparentizing the monitoring and operational tools that depend on /proc file system deployed in container or on the host. Existing monitoring and operational tools will be able to transition into container to achieve in-container monitoring and operations without appropriating structure or re-developing.

Next, let's see from an example of installing PouchContainer 0.3.0 in Ubuntu:


```
# uname -a
Linux p4 4.13.0-36-generic #40~16.04.1-Ubuntu SMP Fri Feb 16 23:25:58 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

systemd invoke pouchd in default mode that does not start LXCFS. Now, the created container cannot use LXCFS functionalities. Let's see the contents under /proc in container:

```
# systemctl start pouch
# head -n 5 /proc/meminfo
MemTotal:        2039520 kB
MemFree:          203028 kB
MemAvailable:     777268 kB
Buffers:          239960 kB
Cached:           430972 kB
root@p4:~# cat /proc/uptime
2594341.81 2208722.33
# pouch run -m 50m -it registry.hub.docker.com/library/busybox:1.28
/ # head -n 5 /proc/meminfo
MemTotal:        2039520 kB
MemFree:          189096 kB
MemAvailable:     764116 kB
Buffers:          240240 kB
Cached:           433928 kB
/ # cat /proc/uptime
2594376.56 2208749.32
```

It is obvious to see the consistency between the outputs from /proc/meminfo、uptime files and those from the host machine. Although we designated 50 M memory in start time, /proc/meminfo files does not demonstrate the memory limit in containers. 

Starting LXCFS service inside the host machine, manually invoking pouchd process and designating related relative LXCFS parameters:

```
# systemctl start lxcfs
# pouchd -D --enable-lxcfs --lxcfs /usr/bin/lxcfs >/tmp/1 2>&1 &
[1] 32707
# ps -ef |grep lxcfs
root       698     1  0 11:08 ?        00:00:00 /usr/bin/lxcfs /var/lib/lxcfs/
root       724 32144  0 11:08 pts/22   00:00:00 grep --color=auto lxcfs
root     32707 32144  0 11:05 pts/22   00:00:00 pouchd -D --enable-lxcfs --lxcfs /usr/bin/lxcfs
```

启动容器，获取相应的文件内容：

```
# pouch run --enableLxcfs -it -m 50m registry.hub.docker.com/library/busybox:1.28
/ # head -n 5 /proc/meminfo
MemTotal:          51200 kB
MemFree:           50804 kB
MemAvailable:      50804 kB
Buffers:               0 kB
Cached:                4 kB
/ # cat /proc/uptime
10.00 10.00
```

Using LXCFS started container and reading in-container /proc files to obtain relative information in container.

### 业务应用
对于大部分对系统依赖较强的应用，应用的启动程序需要获取系统的内存、CPU 等相关信息，从而进行相应的配置。当容器内的 /proc 文件无法准确反映容器内资源的情况，会对上述应用造成不可忽视的影响。

例如对于一些 Java 应用，也存在启动脚本中查看 /proc/meminfo 动态分配运行程序的堆栈大小，当容器内存限制小于宿主机内存时，会发生分配内存失败引起的程序启动失败。对于 DPDK 相关应用，应用工具需要根据 /proc/cpuinfo 获取 CPU 信息，得到应用初始化 EAL 层所使用的 CPU 逻辑核。如果容器内无法准确获取上述信息，对于 DPDK 应用而言，则需要修改相应的工具。

## PouchContainer 集成 LXCFS
PouchContainer 从 0.1.0 版开始即支持 LXCFS，具体实现可以参见: [https://github.com/alibaba/pouch/pull/502](https://github.com/alibaba/pouch/pull/502) .

简而言之，容器启动时，通过-v 将宿主机上 LXCFS 的挂载点 /var/lib/lxc/lxcfs/proc/ 挂载到容器内部的虚拟 /proc 文件系统目录下。此时在容器内部 /proc 目录下可以看到，一些列proc文件，包括 meminfo, uptime, swaps, stat, diskstats, cpuinfo 等。具体使用参数如下：

```
-v /var/lib/lxc/:/var/lib/lxc/:shared
-v /var/lib/lxc/lxcfs/proc/uptime:/proc/uptime 
-v /var/lib/lxc/lxcfs/proc/swaps:/proc/swaps 
-v /var/lib/lxc/lxcfs/proc/stat:/proc/stat 
-v /var/lib/lxc/lxcfs/proc/diskstats:/proc/diskstats 
-v /var/lib/lxc/lxcfs/proc/meminfo:/proc/meminfo 
-v /var/lib/lxc/lxcfs/proc/cpuinfo:/proc/cpuinfo
```

为了简化使用，pouch create 和 run 命令行提供参数 `--enableLxcfs`, 创建容器时指定上述参数，即可省略复杂的 `-v` 参数。

经过一段时间的使用和测试，我们发现由于lxcfs重启之后，会重建proc和cgroup，导致在容器里访问 /proc 出现 `connect failed` 错误。为了增强 LXCFS 稳定性，在 PR:[https://github.com/alibaba/pouch/pull/885](https://github.com/alibaba/pouch/pull/885) 中，refine LXCFS 的管理方式，改由 systemd 保障，具体实现方式为在 lxcfs.service 加上 ExecStartPost 做 remount 操作，并且遍历使用了 LXCFS 的容器，在容器内重新 mount。

## 总结
PouchContainer 支持 LXCFS 实现容器内 /proc 文件系统的视图隔离，将大大减少企业存量应用容器化的过程中原有工具链和运维习惯的改变，加快容器化进度。有力支撑企业从传统虚拟化到容器虚拟化的平稳转型。
