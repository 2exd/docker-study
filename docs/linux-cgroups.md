---
title: linux-cgroups
author: 2exd
date: 2023-03-17 13:25
updated: 2023-03-17 15:19
---

# cgroup

Linux Cgroups（Control Groups）提供了一对进程及将来子进程的资源限制、控制和统计能力，这些资源包括CPU、内存、存储和网络等。通过 cgroups 可以方便地限制某个进程的资源占用，并且可以实时监控进程的信息。

## 1 创建cgroup树

创建并挂载一个 hierarchy (cgroup树)，如下

```shell
# 创建一个hierarchy挂载点
~/docker-study (main) # mkdir cgroup-test                                                                                                                                          root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# 挂在一个hierarchy
~/docker-study (main) # sudo mount -t cgroup -o none,name=cgroup-test cgroup-test ./cgroup-test                                                                                    root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# 挂在后可以看到系统在此目录下生成了一些默认文件
~/docker-study (main*) # ls ./cgroup-test                                                                                                                                          root@VM-4-3-centos
cgroup.clone_children  cgroup.event_control  cgroup.procs  cgroup.sane_behavior  notify_on_release  release_agent  tasks
```

## 2 扩展两个子cgroup

```shell
~/docker-study (main*) # cd cgroup-test                                                                                                                                            root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~/docker-study/cgroup-test # sudo mkdir cgroup-1                                                                                                                                   root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~/docker-study/cgroup-test # sudo mkdir cgroup-2                                                                                                                                   root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~/docker-study/cgroup-test # tree                                                                                                                                                  root@VM-4-3-centos
.
├── cgroup-1
│   ├── cgroup.clone_children
│   ├── cgroup.event_control
│   ├── cgroup.procs
│   ├── notify_on_release
│   └── tasks
├── cgroup-2
│   ├── cgroup.clone_children
│   ├── cgroup.event_control
│   ├── cgroup.procs
│   ├── notify_on_release
│   └── tasks
├── cgroup.clone_children
├── cgroup.event_control
├── cgroup.procs
├── cgroup.sane_behavior
├── notify_on_release
├── release_agent
└── tasks

2 directories, 17 files
```

## 3 添加和移动进程

在 cgroup 中添加和移动进程

```shell
~/docker-study/cgroup-test/cgroup-1 # echo $$                                                                                                                                      root@VM-4-3-centos
2979
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~/docker-study/cgroup-test/cgroup-1 # sudo sh -c "echo $$ >> tasks"                                                                                                                root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~/docker-study/cgroup-test/cgroup-1 # cat /proc/2979/cgroup                                                                                                                        root@VM-4-3-centos
12:name=cgroup-test:/cgroup-1
11:memory:/user.slice
10:blkio:/user.slice
9:cpuset:/
8:net_prio,net_cls:/
7:devices:/user.slice
6:pids:/user.slice
5:perf_event:/
4:freezer:/
3:hugetlb:/
2:cpuacct,cpu:/user.slice
1:name=systemd:/user.slice/user-0.slice/session-409317.scope
```

可以看到，当前进程已经在 `cgroup-test:/cgroup-1` 中了

## 4 限制资源

使用 **==subsystem==** 限制 cgroup 中进程的资源

在上面创建 hierarchy 的时候，没有关联到任何的 subsystem ，所以没办法通过那个 hierarchy 中的 cgroup 节点限制进程的资源占用。其实系统已经默认为每个 subsystem 创建了一个 hierarchy，比如 memory 的 Hierarchy

```shell
~/docker-study/cgroup-test # mount |grep memory                                                                                                                                    root@VM-4-3-centos
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
```

可以看到 /sys/fs/cgroup/memory 目录挂在了 memory subsystem 的 hierarchy 上。

下面，就通过在这个 hierarchy 中创建 cgroup，限制如下进程占用的内存：

```shell
# 先启动一个占用内存的 stress 进程
/sys/fs/cgroup/memory/test-limit-memory # stress --vm-bytes 100m --vm-keep -m 1  
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~/docker-study/cgroup-test/test-limit-memory # cd /sys/fs/cgroup/memory                                                                                                            root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# 创建一个 cgroup
/sys/fs/cgroup/memory # sudo mkdir test-limit-memory && cd test-limit-memory                                                                                                       root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# 设置 croup 最大内存占用
/sys/fs/cgroup/memory/test-limit-memory # sudo sh -c "echo "100m" > memory.limit_in_bytes "                                                                                        root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# 将当前进程移动进这个 cgroup
/sys/fs/cgroup/memory/test-limit-memory # sudo sh -c "echo $$ > tasks"                                                                                                             root@VM-4-3-centos
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# 再次开启一个 stess 进程
/sys/fs/cgroup/memory/test-limit-memory # stress --vm-bytes 100m --vm-keep -m 1                                                                                                    root@VM-4-3-centos
stress: info: [15594] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: FAIL: [15594] (415) <-- worker 15595 got signal 9
stress: WARN: [15594] (417) now reaping child worker processes
stress: FAIL: [15594] (451) failed run completed in 0s
```

在 centos 7.9 和 ubuntu 20.04 并没有成功限制内存使用，而是内存超过限制进程**直接被 kill 了**。

# docker使用cgroup

docker run -m 设置内存限制

```shell
~ # docker run -itd -m 128m -p 32322:32322  docker.io/deluan/navidrome                                                                                                1 ↵ root@VM-4-3-centos
94f11608dee4636a7c1add9b3a5edf4710e44b0df582c907a68ce0ea8d08d867
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~ # docker exec -it musing_clarke sh  
/ # cat /sys/fs/cgroup/memory/memory.limit_in_bytes 
134217728 #128m
```

可以发现 docker 会为容器在系统的 hierarchy 中创建 cgroup

# Go实现

`top -p $pid`

`/proc/self/exe` 它代表当前程序，我们可以用 readlink 读取它的源路径就可以获取当前程序的绝对路径。

```shell
~/docker-study (main*) # go run cgroupdemo.go                                                                                                                             root@VM-4-3-centos
13340
current pid 1
stress: info: [5] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
```

查看进程树

```shell
~/docker-study (main*) # pstree -pl
 ├─sshd(1249)─┬─sshd(2975)───zsh(2979)───go(13299)─┬─cgroupdemo(13336)─┬─exe(13340)─┬─stress(13344)───stress(13345)
           │            │                                    │                   │            ├─{exe}(13341)
           │            │                                    │                   │            ├─{exe}(13342)
           │            │                                    │                   │            └─{exe}(13343)
           │            │                                    │                   ├─{cgroupdemo}(13337)
           │            │                                    │                   ├─{cgroupdemo}(13338)
           │            │                                    │                   └─{cgroupdemo}(13339)
```

```shell
# top -p 13345  
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                  
13345 root      20   0  109716 102500    124 R  99.0  2.6   4:52.45 stress   
```

```shell
12609 
current pid 1 
stress: info: [5] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd 
stress: FAIL: [5] (415) <-- worker 6 got signal 9 
stress: WARN: [5] (417) now reaping child worker processes 
stress: FAIL: [5] (421) kill error: No such process 
stress: FAIL: [5] (451) failed run completed in 0s 2023/03/17 15:07:48 exit status 1 
--------------------------------------
```

再起一个工作进程，但是收到信号 9，随后，该进程无法完成，导致程序退出，状态代码为 1。可以看到并没有限制资源而是直接 kill 了进程。
