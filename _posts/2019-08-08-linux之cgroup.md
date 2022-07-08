---
layout: post
published: true
title:  linux之cgroup
categories: [linux]
tags: [cgroup,linux]
---
* content
{:toc}

# 官网定义及操作
```
cgroups - Linux control groups
DESCRIPTION
Control groups, usually referred to as cgroups, are a Linux kernel
      feature which allow processes to be organized into hierarchical
      groups whose usage of various types of resources can then be limited
      and monitored.  The kernel's cgroup interface is provided through a
      pseudo-filesystem called cgroupfs.  Grouping is implemented in the
      core cgroup kernel code, while resource tracking and limits are
      implemented in a set of per-resource-type subsystems (memory, CPU,
      and so on).
      Terminology
             A cgroup is a collection of processes that are bound to a set of
             limits or parameters defined via the cgroup filesystem.

             A subsystem is a kernel component that modifies the behavior of the
             processes in a cgroup.  Various subsystems have been implemented,
             making it possible to do things such as limiting the amount of CPU
             time and memory available to a cgroup, accounting for the CPU time
             used by a cgroup, and freezing and resuming execution of the
             processes in a cgroup.  Subsystems are sometimes also known as
             resource controllers (or simply, controllers).

             The cgroups for a controller are arranged in a hierarchy.  This
             hierarchy is defined by creating, removing, and renaming
             subdirectories within the cgroup filesystem.  At each level of the
             hierarchy, attributes (e.g., limits) can be defined.  The limits,
             control, and accounting provided by cgroups generally have effect
             throughout the subhierarchy underneath the cgroup where the
             attributes are defined.  Thus, for example, the limits placed on a
             cgroup at a higher level in the hierarchy cannot be exceeded by
             descendant cgroups.

         Cgroups version 1 and version 2
             The initial release of the cgroups implementation was in Linux
             2.6.24.  Over time, various cgroup controllers have been added to
             allow the management of various types of resources.  However, the
             development of these controllers was largely uncoordinated, with the
             result that many inconsistencies arose between controllers and
             management of the cgroup hierarchies became rather complex.  (A
             longer description of these problems can be found in the kernel
             source file Documentation/cgroup-v2.txt.)

             Because of the problems with the initial cgroups implementation
             (cgroups version 1), starting in Linux 3.10, work began on a new,
             orthogonal implementation to remedy these problems.  Initially marked
             experimental, and hidden behind the -o __DEVEL__sane_behavior mount
             option, the new version (cgroups version 2) was eventually made
             official with the release of Linux 4.5.  Differences between the two
             versions are described in the text below.

             Although cgroups v2 is intended as a replacement for cgroups v1, the
             older system continues to exist (and for compatibility reasons is
             unlikely to be removed).  Currently, cgroups v2 implements only a
             subset of the controllers available in cgroups v1.  The two systems
             are implemented so that both v1 controllers and v2 controllers can be
             mounted on the same system.  Thus, for example, it is possible to use
             those controllers that are supported under version 2, while also
             using version 1 controllers where version 2 does not yet support
             those controllers.  The only restriction here is that a controller
             can't be simultaneously employed in both a cgroups v1 hierarchy and
             in the cgroups v2 hierarchy.
```
以上为linux man查出来的英文解释，现在翻译下：
```
描述：
控制组，也叫cgroups，是Linux内核的一个特性，允许将进程组织成分层的组，这些组里不同类型的资源可以被限制和监控。内核的cgroups接口通过一个伪文件系统cgroupfs提供。这些分组被部署在内核代码中，资源的跟踪和权限部署在每个资源类型子系统中，如内存、cpu等。

专业术语：
一个cgroup（控制组）就是一系列的进程的集合，它们绑定在一系列被cgroup filesystem定义的限制或是参数里。
一个subsystem它是内核的组件，它们改变着在cgroup里的进程的行为。内核中有各种各样的子系统，能做各种事情，如限制cpu时间和可用内存、计算被一个cgroup使用的时间，让cgroup里的进程冻结或是继续执行。子系统有时因为资源控制（resource controller）而被人所熟悉（或者简单点要controller）.
在一个子系统中的所有cgroups都被放进了一个等级系统（hierarchy）里。这个系统被定义在通过控制组文件系统（cgroup filesystem）创建、移动和重命名子目录的这些操作里。在这个系统里的每一层，属性（如权限）都可以被定义。由cgroup提供的权限、控制和帐户在它所衍生出的子系统中一直生效。例如在一个高级别的cgroup中定义的限制就不能被它的子cgroups所超过。

cgroup有两个版本：version1和version2
最开始是在linux 2.6.22内核中集成，慢慢地各种各样的控制组控制器（cgroup controllers）慢慢都被加进来管理不同的资源。很多冲突出现了，然后在3.10内核开始重构，这之后的版本一直是开发版本，直到4.5才变成官方稳定版本。它们之间的差异如下：
尽官v2的目的是替代v1，但由于老系统不可能马上废除、兼容性考虑，因此v2仅仅只是v1控制器里的一个子集。这两套系统都部署了，因此这两个版本的控制器都可以挂载到同一套系统。因此，可以在v1里用v2的控制器，也可以使用目前还没有支持的v1的控制器。现在它们最大的限制就是同一个控制器不能同时既部署在v1 的等级系统也在v2的等级系统里。
```

上面是Linux的man手册的的翻译，官网具体详细的介绍可以参考redhat[官网](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/resource_management_guide/index)；
也可以参考ubuntu的[介绍](http://manpages.ubuntu.com/manpages/disco/en/man7/cgroups.7.html)

## 挂载v1 controllers

In  order  to  use  a  v1 controller, it must be mounted against a cgroup filesystem.  The
usual place for such mounts is under a  tmpfs(5)  filesystem  mounted  at  /sys/fs/cgroup.
Thus, one might mount the cpu controller as follows:

    mount -t cgroup -o cpu none /sys/fs/cgroup/cpu

It  is  possible to comount multiple controllers against the same hierarchy.  For example,
here the cpu and cpuacct controllers are comounted against a single hierarchy:

    mount -t cgroup -o cpu,cpuacct none /sys/fs/cgroup/cpu,cpuacct

Comounting controllers has the effect that a process is in the same cgroup for all of  the comounted  controllers.   Separately mounting controllers allows a process to be in cgroup /foo1 for one controller while being in /foo2/foo3 for another.It is possible to comount all v1 controllers against the same hierarchy:

    mount -t cgroup -o all cgroup /sys/fs/cgroup

Note  that  on  many  systems,  the  v1  controllers  are  automatically   mounted   under  /sys/fs/cgroup; in particular, systemd(1) automatically creates such mount points.

## Cgroups version 1 controllers

Each of the cgroups version 1 controllers is governed by  a  kernel  configuration  option
(listed  below).  Additionally, the availability of the cgroups feature is governed by the
CONFIG_CGROUPS kernel configuration option.
[参考](http://manpages.ubuntu.com/manpages/disco/en/man7/cgroups.7.html)

## Creating cgroups and moving processes
A cgroup filesystem initially contains a single root  cgroup,  '/',  which  all  processes
belong to.  A new cgroup is created by creating a directory in the cgroup filesystem:

mkdir /sys/fs/cgroup/cpu/cg1

This creates a new empty cgroup.

A  process  may  be moved to this cgroup by writing its PID into the cgroup's cgroup.procs
file:

echo $$ > /sys/fs/cgroup/cpu/cg1/cgroup.procs

Only one PID at a time should be written to this file.

Writing the value 0 to a cgroup.procs file causes the writing process to be moved  to  the
corresponding cgroup.

When  writing  a  PID into the cgroup.procs, all threads in the process are moved into the
new cgroup at once.

##   Removing cgroups
To remove a cgroup, it must first  have  no  child  cgroups  and  contain  no  (nonzombie)
processes.  So long as that is the case, one can simply remove the corresponding directory
pathname.  Note that files in a cgroup directory cannot and need not be removed.


## /proc/cgroups及相关的几个文件的介绍

```  
/proc files
    /proc/cgroups (since Linux 2.6.24)
           This  file  contains  information  about the controllers that are compiled into the
           kernel.  An example of the contents of this file (reformatted for  readability)  is
           the following:


                  #subsys_name    hierarchy      num_cgroups    enabled
                  cpuset          4              1              1
                  cpu             8              1              1
                  cpuacct         8              1              1
                  blkio           6              1              1
                  memory          3              1              1
                  devices         10             84             1
                  freezer         7              1              1
                  net_cls         9              1              1
                  perf_event      5              1              1
                  net_prio        9              1              1
                  hugetlb         0              1              0
                  pids            2              1              1
                  The fields in this file are, from left to right:

                  1. The name of the controller.

                  2. The  unique  ID of the cgroup hierarchy on which this controller is mounted.  If
                     multiple cgroups v1 controllers are bound to the same hierarchy, then each  will
                     show the same hierarchy ID in this field.  The value in this field will be 0 if:

                       a) the controller is not mounted on a cgroups v1 hierarchy;

                       b) the controller is bound to the cgroups v2 single unified hierarchy; or

                       c) the controller is disabled (see below).

                  3. The number of control groups in this hierarchy using this controller.

                  4. This  field  contains  the value 1 if this controller is enabled, or 0 if it has
                     been disabled (via the cgroup_disable kernel command-line boot parameter).

/proc/[pid]/cgroup (since Linux 2.6.24)
             This file describes control groups to which the process with the corresponding  PID
             belongs.   The  displayed  information  differs for cgroups version 1 and version 2
             hierarchies.

             For each cgroup hierarchy of which the process is a  member,  there  is  one  entry
             containing three colon-separated fields:

                 hierarchy-ID:controller-list:cgroup-path

             For example:

                 5:cpuacct,cpu,cpuset:/daemons

             The colon-separated fields are, from left to right:

             1. For  cgroups  version  1  hierarchies, this field contains a unique hierarchy ID
                number that can be matched to a hierarchy ID in /proc/cgroups.  For the  cgroups
                version 2 hierarchy, this field contains the value 0.

             2. For cgroups version 1 hierarchies, this field contains a comma-separated list of
                the controllers bound to the hierarchy.  For the cgroups  version  2  hierarchy,
                this field is empty.

             3. This  field contains the pathname of the control group in the hierarchy to which
                the process belongs.  This pathname is  relative  to  the  mount  point  of  the
                hierarchy.

  /sys/kernel/cgroup files
      /sys/kernel/cgroup/delegate (since Linux 4.15)
             This  file  exports  a  list  of  the  cgroups  v2  files  (one  per line) that are
             delegatable (i.e., whose ownership  should  be  changed  to  the  user  ID  of  the
             delegatee).   In  the  future, the set of delegatable files may change or grow, and
             this file provides a way for the kernel to inform user-space applications of  which
             files  must be delegated.  As at Linux 4.15, one sees the following when inspecting
             this file:

                 $ cat /sys/kernel/cgroup/delegate
                 cgroup.procs
                 cgroup.subtree_control
                 cgroup.threads

      /sys/kernel/cgroup/features (since Linux 4.15)
             Over time, the set of cgroups v2 features that  are  provided  by  the  kernel  may
             change or grow, or some features may not be enabled by default.  This file provides
             a way for user-space applications to discover  what  features  the  running  kernel
             supports and has enabled.  Features are listed one per line:

                 $ cat /sys/kernel/cgroup/features
                 nsdelegate

             The entries that can appear in this file are:

             nsdelegate (since Linux 4.15)
                    The kernel supports the nsdelegate mount option.
```

## 功能
1. Resource limitation: 限制资源使用，比如内存使用上限以及文件系统的缓存限制。  
2. Prioritization: 优先级控制，比如：CPU利用和磁盘IO吞吐。  
3. Accounting: 一些审计或一些统计，主要目的是为了计费。  
4. Control: 挂起进程，恢复执行进程。  

使​​​用​​​ cgroup，系​​​统​​​管​​​理​​​员​​​可​​​更​​​具​​​体​​​地​​​控​​​制​​​对​​​系​​​统​​​资​​​源​​​的​​​分​​​配​​​、​​​优​​​先​​​顺​​​序​​​、​​​拒​​​绝​​​、​​​管​​​理​​​和​​​监​​​控​​​。​​​可​​​更​​​好​​​地​​​根​​​据​​​任​​​务​​​和​​​用​​​户​​​分​​​配​​​硬​​​件​​​资​​​源​​​，提​​​高​​​总​​​体​​​效​​​率​​​

在实践中，系统管理员一般会利用CGroup做下面这些事（有点像为某个虚拟机分配资源似的）：
```
隔离一个进程集合（比如：nginx的所有进程），并限制他们所消费的资源，比如绑定CPU的核。
为这组进程 分配其足够使用的内存
为这组进程分配相应的网络带宽和磁盘存储限制
限制访问某些设备（通过设置设备的白名单）
```

首先，Linux把CGroup这个事实现成了一个file system，你可以mount

```
root@default:~/qy# mount -t cgroup
systemd on /sys/fs/cgroup/systemd type cgroup (rw,noexec,nosuid,nodev,none,name=systemd)

或者使用lssubsys
root@default:~/qy# lssubsys -m
cpuset /sys/fs/cgroup/cpuset
cpu /sys/fs/cgroup/cpu
cpuacct /sys/fs/cgroup/cpuacct
memory /sys/fs/cgroup/memory
devices /sys/fs/cgroup/devices
freezer /sys/fs/cgroup/freezer
blkio /sys/fs/cgroup/blkio
perf_event /sys/fs/cgroup/perf_event
hugetlb /sys/fs/cgroup/hugetlb

root@default:~/qy# lscgroup
cpuset:/
cpuset:/user
cpuset:/user/1000.user
cpuset:/user/1000.user/3.session
cpu:/
cpu:/user
cpu:/user/1000.user
cpu:/user/1000.user/3.session
cpuacct:/
cpuacct:/user
cpuacct:/user/1000.user
cpuacct:/user/1000.user/3.session
memory:/
memory:/user
memory:/user/1000.user
memory:/user/1000.user/3.session
devices:/
devices:/user
devices:/user/1000.user
devices:/user/1000.user/3.session
freezer:/
freezer:/user
freezer:/user/1000.user
freezer:/user/1000.user/3.session
blkio:/
blkio:/user
blkio:/user/1000.user
blkio:/user/1000.user/3.session
perf_event:/
perf_event:/user
perf_event:/user/1000.user
perf_event:/user/1000.user/3.session
hugetlb:/
hugetlb:/user
hugetlb:/user/1000.user
hugetlb:/user/1000.user/3.session

root@default:/proc# lssubsys
cpuset
cpu
cpuacct
memory
devices
freezer
blkio
perf_event
hugetlb
```
我们可以看到，在/sys/fs下有一个cgroup的目录，这个目录下还有很多子目录，比如： cpu，cpuset，memory，blkio……这些，这些都是cgroup的子系统。分别用于干不同的事的。

如果你没有看到上述的目录，你可以自己mount，下面给了一个示例：

```
mkdir cgroup
mount -t tmpfs cgroup_root ./cgroup
mkdir cgroup/cpuset
mount -t cgroup -ocpuset cpuset ./cgroup/cpuset/
mkdir cgroup/cpu
mount -t cgroup -ocpu cpu ./cgroup/cpu/
mkdir cgroup/memory
mount -t cgroup -omemory memory ./cgroup/memory/
```
一旦mount成功，你就会看到这些目录下就有好文件了，比如，如下所示的cpu和cpuset的子系统：
```
$ ls /sys/fs/cgroup/cpu /sys/fs/cgroup/cpuset/
/sys/fs/cgroup/cpu:
cgroup.clone_children  cgroup.sane_behavior  cpu.shares         release_agent
cgroup.event_control   cpu.cfs_period_us     cpu.stat           tasks
cgroup.procs           cpu.cfs_quota_us      notify_on_release  user

/sys/fs/cgroup/cpuset/:
cgroup.clone_children  cpuset.mem_hardwall             cpuset.sched_load_balance
cgroup.event_control   cpuset.memory_migrate           cpuset.sched_relax_domain_level
cgroup.procs           cpuset.memory_pressure          notify_on_release
cgroup.sane_behavior   cpuset.memory_pressure_enabled  release_agent
cpuset.cpu_exclusive   cpuset.memory_spread_page       tasks
cpuset.cpus            cpuset.memory_spread_slab       user
cpuset.mem_exclusive   cpuset.mems
```

你可以到/sys/fs/cgroup的各个子目录下去make个dir，你会发现，一旦你创建了一个子目录，这个子目录里又有很多文件了。
```
@ubuntu:/sys/fs/cgroup/cpu$ sudo mkdir haoel
[sudo] password for hchen:
hchen@ubuntu:/sys/fs/cgroup/cpu$ ls ./haoel
cgroup.clone_children  cgroup.procs       cpu.cfs_quota_us  cpu.stat           tasks
cgroup.event_control   cpu.cfs_period_us  cpu.shares        notify_on_release
```
好了，我们来看几个示例。

## CPU 限制

假设，我们有一个非常吃CPU的程序，叫deadloop，其源码如下：

DEADLOOP.C
```
int main(void)
{
    int i = 0;
    for(;;) i++;
    return 0;
}
```
用sudo执行起来后，毫无疑问，CPU被干到了100%（下面是top命令的输出）

```
PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND     
%Cpu(s): 99.7 us,  0.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 s
```
然后，我们这前不是在/sys/fs/cgroup/cpu下创建了一个zk的group。直接mkdir zk我们会发现这个目录下面生成了一系列的相关文件。
```
root@default:/sys/fs/cgroup/cpu/zk# ls
cgroup.clone_children  cpu.cfs_period_us  cpu.stat
cgroup.event_control   cpu.cfs_quota_us   notify_on_release
cgroup.procs           cpu.shares         tasks
```
我们先设置一下这个group的cpu利用的限制：
```
root@default:/sys/fs/cgroup/cpu/zk# cat cpu.cfs_quota_us
-1
root@default:/sys/fs/cgroup/cpu/zk# echo 20000 > cpu.cfs_quota_us
```
我们看到，这个进程的PID是3946，我们把这个进程加到这个cgroup中：

```
# echo 3946 >> /sys/fs/cgroup/cpu/zk/tasks
```
然后，就会在top中看到CPU的利用立马下降成20%了。（前面我们设置的20000就是20%的意思）

```
PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND     
 3946 root      20   0    4192    352    280 R 20.0  0.1   1:14.02
 ```

下面的代码是一个线程的示例：

```
#define _GNU_SOURCE         /* See feature_test_macros(7) */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/syscall.h>


const int NUM_THREADS = 5;

void *thread_main(void *threadid)
{
    /* 把自己加入cgroup中（syscall(SYS_gettid)为得到线程的系统tid） */
    char cmd[128];
    sprintf(cmd, "echo %ld >> /sys/fs/cgroup/cpu/haoel/tasks", syscall(SYS_gettid));
    system(cmd);
    sprintf(cmd, "echo %ld >> /sys/fs/cgroup/cpuset/haoel/tasks", syscall(SYS_gettid));
    system(cmd);

    long tid;
    tid = (long)threadid;
    printf("Hello World! It's me, thread #%ld, pid #%ld!\n", tid, syscall(SYS_gettid));

    int a=0;
    while(1) {
        a++;
    }
    pthread_exit(NULL);
}
int main (int argc, char *argv[])
{
    int num_threads;
    if (argc > 1){
        num_threads = atoi(argv[1]);
    }
    if (num_threads<=0 || num_threads>=100){
        num_threads = NUM_THREADS;
    }

    /* 设置CPU利用率为50% */
    mkdir("/sys/fs/cgroup/cpu/haoel", 755);
    system("echo 50000 > /sys/fs/cgroup/cpu/haoel/cpu.cfs_quota_us");

    mkdir("/sys/fs/cgroup/cpuset/haoel", 755);
    /* 限制CPU只能使用#2核和#3核 */
    system("echo \"2,3\" > /sys/fs/cgroup/cpuset/haoel/cpuset.cpus");

    pthread_t* threads = (pthread_t*) malloc (sizeof(pthread_t)*num_threads);
    int rc;
    long t;
    for(t=0; t<num_threads; t++){
        printf("In main: creating thread %ld\n", t);
        rc = pthread_create(&threads[t], NULL, thread_main, (void *)t);
        if (rc){
            printf("ERROR; return code from pthread_create() is %d\n", rc);
            exit(-1);
        }
    }

    /* Last thing that main() should do */
    pthread_exit(NULL);
    free(threads);
}
```

## 内存使用限制

我们再来看一个限制内存的例子（下面的代码是个死循环，其它不断的分配内存，每次512个字节，每次休息一秒）：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>

int main(void)
{
    int size = 0;
    int chunk_size = 512;
    void *p = NULL;

    while(1) {

        if ((p = malloc(p, chunk_size)) == NULL) {
            printf("out of memory!!\n");
            break;
        }
        memset(p, 1, chunk_size);
        size += chunk_size;
        printf("[%d] - memory is allocated [%8d] bytes \n", getpid(), size);
        sleep(1);
    }
    return 0;
}
```

然后，在我们另外一边：

```
# 创建memory cgroup
$ mkdir /sys/fs/cgroup/memory/haoel
$ echo 64k > /sys/fs/cgroup/memory/haoel/memory.limit_in_bytes

# 把上面的进程的pid加入这个cgroup
$ echo [pid] > /sys/fs/cgroup/memory/haoel/tasks
```
你会看到，一会上面的进程就会因为内存问题被kill掉了。

## 磁盘I/O限制
我们先看一下我们的硬盘IO，我们的模拟命令如下：（从/dev/sda1上读入数据，输出到/dev/null上）

```
sudo dd if=/dev/sda1 of=/dev/null
```
我们通过iotop命令我们可以看到相关的IO速度是55MB/s（虚拟机内）：
```
TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND          
8128 be/4 root       55.74 M/s    0.00 B/s  0.00 % 85.65 % dd if=/de~=/dev/null...
```
然后，我们先创建一个blkio（块设备IO）的cgroup

```
mkdir /sys/fs/cgroup/blkio/haoel
```
并把读IO限制到1MB/s，并把前面那个dd命令的pid放进去（注：8:0 是设备号，你可以通过ls -l /dev/sda1获得）：
```
root@ubuntu:~# echo '8:0 1048576'  > /sys/fs/cgroup/blkio/haoel/blkio.throttle.read_bps_device
root@ubuntu:~# echo 8128 > /sys/fs/cgroup/blkio/haoel/tasks
```
再用iotop命令，你马上就能看到读速度被限制到了1MB/s左右。

```
TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND          
8128 be/4 root      973.20 K/s    0.00 B/s  0.00 % 94.41 % dd if=/de~=/dev/null...
```

## CGroup的子系统
好了，有了以上的感性认识我们来，我们来看看control group有哪些子系统：

+ blkio — 这​​​个​​​子​​​系​​​统​​​为​​​块​​​设​​​备​​​设​​​定​​​输​​​入​​​/输​​​出​​​限​​​制​​​，比​​​如​​​物​​​理​​​设​​​备​​​（磁​​​盘​​​，固​​​态​​​硬​​​盘​​​，USB 等​​​等​​​）。
+ cpu — 这​​​个​​​子​​​系​​​统​​​使​​​用​​​调​​​度​​​程​​​序​​​提​​​供​​​对​​​ CPU 的​​​ cgroup 任​​​务​​​访​​​问​​​。​​​
+ cpuacct — 这​​​个​​​子​​​系​​​统​​​自​​​动​​​生​​​成​​​ cgroup 中​​​任​​​务​​​所​​​使​​​用​​​的​​​ CPU 报​​​告​​​。​​​
+ cpuset — 这​​​个​​​子​​​系​​​统​​​为​​​ cgroup 中​​​的​​​任​​​务​​​分​​​配​​​独​​​立​​​ CPU（在​​​多​​​核​​​系​​​统​​​）和​​​内​​​存​​​节​​​点​​​。​​​
+ devices — 这​​​个​​​子​​​系​​​统​​​可​​​允​​​许​​​或​​​者​​​拒​​​绝​​​ cgroup 中​​​的​​​任​​​务​​​访​​​问​​​设​​​备​​​。​​​
+ freezer — 这​​​个​​​子​​​系​​​统​​​挂​​​起​​​或​​​者​​​恢​​​复​​​ cgroup 中​​​的​​​任​​​务​​​。​​​
+ memory — 这​​​个​​​子​​​系​​​统​​​设​​​定​​​ cgroup 中​​​任​​​务​​​使​​​用​​​的​​​内​​​存​​​限​​​制​​​，并​​​自​​​动​​​生​​​成​​​​​内​​​存​​​资​​​源使用​​​报​​​告​​​。​​​
+ net_cls — 这​​​个​​​子​​​系​​​统​​​使​​​用​​​等​​​级​​​识​​​别​​​符​​​（classid）标​​​记​​​网​​​络​​​数​​​据​​​包​​​，可​​​允​​​许​​​ Linux 流​​​量​​​控​​​制​​​程​​​序​​​（tc）识​​​别​​​从​​​具​​​体​​​ cgroup 中​​​生​​​成​​​的​​​数​​​据​​​包​​​。​​​
+ net_prio — 这个子系统用来设计网络流量的优先级
+ hugetlb — 这个子系统主要针对于HugeTLB系统进行限制，这是一个大页文件系统。

注意，你可能在Ubuntu 14.04下看不到net_cls和net_prio这两个cgroup，你需要手动mount一下：

```
$ sudo modprobe cls_cgroup
$ sudo mkdir /sys/fs/cgroup/net_cls
$ sudo mount -t cgroup -o net_cls none /sys/fs/cgroup/net_cls

$ sudo modprobe netprio_cgroup
$ sudo mkdir /sys/fs/cgroup/net_prio
$ sudo mount -t cgroup -o net_prio none /sys/fs/cgroup/net_prio
```
关于各个子系统的参数细节，以及更多的Linux CGroup的文档，你可以看看下面的文档：

# CGroup的术语
CGroup有下述术语：

+  任务（Tasks）：就是系统的一个进程。
+ 控制组（Control Group）：一组按照某种标准划分的进程，比如官方文档中的Professor和Student，或是WWW和System之类的，其表示了某进程组。Cgroups中的资源控制都是以控制组为单位实现。一个进程可以加入到某个控制组。而资源的限制是定义在这个组上，就像上面示例中我用的haoel一样。简单点说，cgroup的呈现就是一个目录带一系列的可配置文件。
+ 层级（Hierarchy）：控制组可以组织成hierarchical的形式，既一颗控制组的树（目录结构）。控制组树上的子节点继承父结点的属性。简单点说，hierarchy就是在一个或多个子系统上的cgroups目录树。
+ 子系统（Subsystem）：一个子系统就是一个资源控制器，比如CPU子系统就是控制CPU时间分配的一个控制器。子系统必须附加到一个层级上才能起作用，一个子系统附加到某个层级以后，这个层级上的所有控制族群都受到这个子系统的控制。Cgroup的子系统可以有很多，也在不断增加中。

# 注意
cgroup 中的 blkio 只能限制整个disk，例如:
“8:0 limitation”，如果写入 “8:1 limitation” 会报错：no such device。

github上对应 kernel 的源码地址：
https://github.com/torvalds/linux/blob/v4.4/block/blk-cgroup.c#L804-L809


另外这个也很详细 https://mp.weixin.qq.com/s/Qg_FvLBzudYa8OH79e1D7Q
这个参考  https://coolshell.cn/articles/17049.html
