---
id: ce380b41-afbc-4b05-915e-b57b91b66a09
title: linux-namespace
author: 2exd
date: 2023-03-17 12:31
updated: 2023-03-17 13:25
---

#NameSpace

# hostname

**==CLONE_NEWUTS==**

## 查看进程树

Linux **==pstree==**(英文全称：display a tree of processes）) 命令将所有进程以树状图显示，树状图将会以 pid (如果有指定) 或是以 init 这个基本进程为根 (root)，如果有指定使用者 id，则树状图会只显示该使用者所拥有的进程。

```shell
-> # go run main.go
# pstree -pl
init(Ubuntu-20.(1)─┬─SessionLeader(10)───zsh(11)
                   ├─SessionLeader(197)───fsnotifier-wsl(198)
                   ├─SessionLeader(3618)───zsh(3619)───go(6060)─┬─main(6193)─┬─sh(6197)───pstree(6262)
                   │                                            │            ├─{main}(6194)
                   │                                            │            ├─{main}(6195)
                   │                                            │            └─{main}(6196)
                   │                                            ├─{go}(6061)
                   │                                            ├─{go}(6062)
                   │                                            ├─{go}(6063)
                   │                                            ├─{go}(6064)
                   │                                            ├─{go}(6065)
                   │                                            ├─{go}(6066)
                   │                                            ├─{go}(6067)
                   │                                            ├─{go}(6068)
                   │                                            ├─{go}(6069)
                   │                                            ├─{go}(6070)
                   │                                            ├─{go}(6071)
                   │                                            ├─{go}(6072)
                   │                                            ├─{go}(6073)
                   │                                            ├─{go}(6074)
                   │                                            ├─{go}(6075)
                   │                                            ├─{go}(6076)
                   │                                            ├─{go}(6077)
                   │                                            ├─{go}(6079)
                   │                                            └─{go}(6173)
                   ├─SessionLeader(5916)───zsh(5917)
                   ├─init(6)───{init}(7)
                   └─{init(Ubuntu-20.}(8)
```

## 验证父子进程是否在同一个UTS Namespace里

**==readlink /proc/$pid/ns/uts==**

```
root@ZEXD [12:27:58 PM] [~]
-> # readlink /proc/6193/ns/uts
uts:[5]
root@ZEXD [12:28:35 PM] [~]
-> # readlink /proc/6197/ns/uts
uts:[5091]
```

确实不在同一个 UTS Namespace 中

## 修改hostname

在 sh 环境里修改 hostname
**==hostname -b bird==**

```shell
# hostname -b bird
# hostname
bird
```

宿主机并没有被修改

```shell
root@ZEXD [12:28:48 PM] [~]
-> # hostname
ZEXD
```

# IPC

**==CLONE_NEWIPC==**

# PID

**==CLONE_NEWPID==**

首先在宿主机看一下进程树

```shell
root@ZEXD [12:46:45 PM] [~/gocode/docker-study] [main *]
-> # pstree -pl
init(Ubuntu-20.(1)─┬─SessionLeader(10)───zsh(11)
                   ├─SessionLeader(197)───fsnotifier-wsl(198)
                   ├─SessionLeader(3618)───zsh(3619)───go(15863)─┬─main(15992)─┬─sh(15997)
                   │                                             │             ├─{main}(15993)
                   │                                             │             ├─{main}(15994)
                   │                                             │             ├─{main}(15995)
                   │                                             │             └─{main}(15996)
                   │                                             ├─{go}(15864)
                   │                                             ├─{go}(15865)
                   │                                             ├─{go}(15866)
                   │                                             ├─{go}(15867)
                   │                                             ├─{go}(15868)
                   │                                             ├─{go}(15869)
                   │                                             ├─{go}(15870)
                   │                                             ├─{go}(15871)
                   │                                             ├─{go}(15872)
                   │                                             ├─{go}(15873)
                   │                                             ├─{go}(15874)
                   │                                             ├─{go}(15875)
                   │                                             ├─{go}(15876)
                   │                                             ├─{go}(15877)
                   │                                             ├─{go}(15879)
                   │                                             ├─{go}(15952)
                   │                                             ├─{go}(15963)
                   │                                             └─{go}(15972)
                   ├─SessionLeader(16372)───zsh(16373)───pstree(16548)
                   ├─init(6)─┬─{init}(7)
                   │         └─{init}(16541)
                   └─{init(Ubuntu-20.}(8)
```

go main 程序运行的 pid 为 **15992**

sh 执行 `echo $$`

```shell
# echo $$
1
```

可以看到打印的 pid 为 1。

```ad-caution
> 但这里还不能使用 ps 或 top 等指令来查看，因为这些指令会使用 /proc 内容
```

# Mount

**==CLONE_NEWNS==**

在 mount namespace 中调用 mount() 和 umount() 仅仅会影响当前 namespace 内的文件系统，对全局的文件系统是没有影响的。

首先运行代码，然后查看 /proc 的文件内容。proc 是一个文件系统，提供额外的机制，可以通过内核和内核模块将信息发送给进程。

```shell
# ls /proc
1   16372  19221  197   3619  cgroups  filesystems  meminfo  self  tty      version_signature
10  16373  19226  198   6     cmdline  interrupts   mounts   stat  uptime
11  19092  19227  3618  bus   cpuinfo  loadavg      net      sys   version
```

因为这里的 proc 还是宿主机的，所以看到会比较乱，下面，将 /proc mount 到我们自己的 namespace 来：
**mount -t proc proc /proc** 用于挂载 Linux 系统外的文件。

```shell
# mount -t proc proc /proc
# ls /proc
1  bus      cmdline  filesystems  loadavg  mounts  self  sys  uptime   version_signature
9  cgroups  cpuinfo  interrupts   meminfo  net     stat  tty  version
```

文件明显少了很多，下面就可以看看系统的进程了

```shell
# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 13:00 tty4     00:00:00 sh
root         8     1  0 13:03 tty4     00:00:00 ps -ef
```

可以看到，在当前的 namespace 中，sh 进程的 pid 为 1。说明了当前的 mount namespace 中的 mount 和外部空间是隔离的， mount 操作并没有影响到外部。_**==docker volume 也是利用了这个特性==**_

# User

**==CLONE_NEWUSER==**

用于隔离用户的用户组id。也就是说，一个进程的 user id 和 group id 在 user namespace 内外是可以不同的。

```shell
# 之前
# id
uid=0(root) gid=0(root) groups=0(root)
###################################################
# 之后
-> # go run main.go
$ id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

# Network

**==CLONE_NEWNET==**

首先在宿主机上查看自己的网卡

```shell
-> # ifconfig
.......................
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 1500
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0xfe<compat,link,site,host>
        loop  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wifi0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.20.205.207  netmask 255.255.0.0  broadcast 10.20.255.255
        inet6 2409:8760:1e81:10::3088  prefixlen 128  scopeid 0x0<global>
        inet6 fe80::d34e:2930:d63:d50a  prefixlen 64  scopeid 0xfd<compat,link,site,host>
        ether b4:0e:de:12:5a:de  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

可以看到，宿主机上有 lo， eth1，wifi0 等设备。

下面运行程序去 network namespace 里看看

```shell
-> # go run main.go
# ifconfig
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 1500
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0xfe<compat,link,site,host>
        loop  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

发现除了 lo 没有其它的网络设备。可以说明 network namespace 与宿主机之间的网络是处于隔离状态的。
