---
layout:      post
title: "Greenplum进程优先级"
subtitle:  ""
excerpt:     ""
author: "jigong"
date: 2019-01-31T10:20:27+08:00
description: "GP集群资源使用率大部分时间是比较低的，而服务器配置的资源较高，这样就造成资源浪费，现在通过混部的模式，在GP集群上使用docker混部storm、nodemanager等计算任务，充分利用集群资源；但是混部后发现GP性能下降较多，发现GP查询进程的nice值高，优先级低。因此需要调整GP集群进程的nice值，来优化混部部署。"
image:       ""
published: true 
tags: 
    - greenplum
    - nice
    - 混合部署
categories:  ["TIPS" ]
---

# 一、GP集群混部
目前GP集群资源使用率大部分时间是比较低的，而服务器配置的资源较高，这样就造成资源浪费，现可以通过混部的模式，在GP集群上使用docker混部storm、nodemanager等应用，充分使用集群资源，但是混部在集群资源紧张、进程较多的时候会影响GP的使用，导致大量的时间都在进程切换，cpu的nice值增高。因此需要调整GP集群进程的nice值，来优化GP集群混部。
# 二、进程优先级说明
* PRI ：Priority 值叫动态优先级，是进程在内核中实际的优先级值，进程优先级的取值范围是通过一个宏定义的，这个宏的名称是MAX_PRIO，它的值为140。Linux实际上实现了140个优先级范围，取值范围是从0-139，这个值越小，优先级越高。0-99表示实时进程，100-139表示普通进程，他们采用的调度算法是不一样的，实时进程更容易获取cpu时间片。普通进程的动态优先级可以通过静态优先级计算获得，动态优先级=max(100,min(静态优先级-bonus+5,139))，其中bonus则是惩罚因子，依赖进程过去运行的情况和进程平均睡眠时间相关，如果小于5，则表示降低动态优先级，大于5则增加动态优先级以示嘉奖。
* NI ：进程Nice值，代表这个进程的优先值;取值范围-20~19，调整该值越低，静态优先级越高，获取的cpu时间片就越长
* 静态优先级：静态优先级的取值范围也是[0-139]，其中 PR 中的 100 to 139 值部分有这么一个对应 PR = 20 + (-20 to +19)，这里的 -20 to +19 便是nice值，所以说两个虽然都是优先级，而且有千丝万缕的关系，但是他们的值，他们的作用范围并不相同。

PRI是比较好理解的，即进程的优先级，或者通俗点说就是程序被CPU执行的先后顺序，此值越小进程的优先级别越高。那NI呢？就是我们所要说的nice值了，其表示进程可被执行的优先级的修正数值。如前面所说，PRI值越小越快被执行，那么加入nice值后，将会使得PRI变为：PRI(new)=PRI(old)+nice。由此看出，PR是根据NICE排序的，规则是NICE越小PR越前（小，优先权更大），即其优先级会变高，则其越快被执行。如果NICE相同则进程uid是root的优先权更大。在LINUX系统中，Nice值的范围从-20到+19（不同系统的值范围是不一样的），正值表示低优先级，负值表示高优先级，值为零则表示不会调整该进程的优先级。具有最高优先级的程序，其nice值最低，所以在LINUX系统中，值-20使得一项任务变得非常重要；与之相反，如果任务的nice为+19，则表示它是一个高尚的、无私的任务，允许所有其他任务比自己享有宝贵的CPU时间的更大使用份额，这也就是nice的名称的来意。进程在创建时被赋予不同的优先级值，而如前面所说，nice的值是表示进程优先级值可被修正数据值，因此，每个进程都在其计划执行时被赋予一个nice值，这样系统就可以根据系统的资源以及具体进程的各类资源消耗情况，主动干预进程的优先级值。在通常情况下，子进程会继承父进程的nice值，比如在系统启动的过程中，init进程会被赋予0，其他所有进程继承了这个nice值（因为其他进程都是init的子进程）。对nice值一个形象比喻，假设在一个CPU轮转中，有2个runnable的进程A和B，如果他们的nice值都为0，假设内核会给他们每人分配1k个cpu时间片。但是假设进程A的为0，但是B的值为-10，那么此时CPU可能分别给A和B分配1k和1.5k的时间片。故可以形象的理解为，nice的值影响了内核分配给进程的cpu时间片的多少，时间片越多的进程，其优先级越高，其优先级值（PRI）越低。%nice，就是改变过优先级的进程的占用CPU的百分比，如上例中就是0.5k/2.5k=1/5=20%。
由此可见，进程nice值和进程优先级不是一个概念，但是进程nice值会影响到进程的优先级变化。

# 三、进程优先级查看
查看进程优先级的方法：
* top命令
使用top命令可以查看到PR和NI选项
   * PR 取值范围为[0,39]表示用户可通过nice值调整那部分区域，即[100,139]，表示的是静态优先级范围，如果该值显示为rt则表示为实时进程
* ps命令
```
ps -afxo user,ppid,pid,stat,pri,ni,time,command
```
pa相关参数说明：
* a 显示所有进程
* -a 显示同一终端下的所有程序
* -A 显示所有进程
* c 显示进程的真实名称
* -N 反向选择
* -e 等于“-A”
* e 显示环境变量
* f 显示程序间的关系
* -H 显示树状结构
* r 显示当前终端的进程
* T 显示当前终端的所有程序
* u 指定用户的所有进程
* -au 显示较详细的资讯
* -aux 显示所有包含其他使用者的行程 
* -C<命令> 列出指定命令的状况
* --lines<行数> 每页显示的行数
* --width<字符数> 每页显示的字符数
* --help 显示帮助信息
* --version 显示版本显示

# 四、调整程序nice值
## 1、执行程序调整nice值
执行该命令的时候需要有root权限，否则无法执行成功。
```
#command 为程序的正常启动命令
nice -n -10 [command]
```
## 2、调整已经启动程序的nice值
```
#调整某个进程的优先级
sudo renice -10 -p [pid]
#调整某个用户进程的优先级
sudo renice -10 -u gpadmin
#调整某个群组进程的优先级
sudo renice -10 -g gpadmin
```
## 3、调整用户默认的nice值
编辑/etc/security/limits.conf配置文件，添加如下内容：
```
gpadmin soft priority -20
```
表示gpadmin用户默认的nice值为-20，那么在gpadmin用户创建的进程nice值也均为-20，修改配置文件，重新登入即可生效。

# 五、调整GP集群的nice值
## 1、参数说明
GP集群提供gp_segworker_relative_priority来配置nice值，其默认值为20，gp_segworker_relative_priority的取值范围为[0-20]，一定要注意设置该值，超出范围将导致集群无法启动，也无法进入master-only模式。该值可以用来调整查询进程的nice值。默认情况下gp启动进程的nice值根据用户默认的nice值而设定，该参数只影响GP的查询进程，会在父进程的nice值上加上该值，赋值给当前进程。因此在不调整nice值，默认情况下，我们会发现查询进程的nice值都很高，达到nice的最大值，而进程优先级最低。
>gpadmin 1 3953970 Ss 19 0 00:04:45 /opt/greenplum/greenplum-db-5.8.0/bin/postgres -D /data/primary1/gpseg128 -p 19900 --gp_dbid=130 --gp_num_contents_in_cluster=152 --silent-mode=true -i -M quiescent --gp_contentid=128
gpadmin 3953970 3953987 Ss 19 0 00:00:55 \_ postgres: 19900, logger process   
gpadmin 3953970 3954138 S 19 0 00:00:32 \_ postgres: 19900, primary process   
gpadmin 3954138 3954153 S 19 0 00:02:28 | \_ postgres: 19900, primary receiver ack process   
gpadmin 3954138 3954154 S 19 0 00:20:07 | \_ postgres: 19900, primary sender process   
gpadmin 3954138 3954155 S 19 0 00:01:59 | \_ postgres: 19900, primary consumer ack process   
gpadmin 3954138 3954156 S 19 0 00:12:37 | \_ postgres: 19900, primary recovery process   
gpadmin 3953970 3954183 Ss 19 0 00:01:16 \_ postgres: 19900, stats collector process   
gpadmin 3953970 3954184 Ss 19 0 00:07:56 \_ postgres: 19900, writer process   
gpadmin 3953970 3954185 Ss 19 0 00:11:04 \_ postgres: 19900, checkpointer process   
gpadmin 3953970 3954186 S 19 0 00:01:44 \_ postgres: 19900, sweeper process   
gpadmin 3953970 3954187 S 19 0 00:21:18 \_ postgres: 19900, stats sender process   
gpadmin 3953970 3954188 Ss 19 0 00:05:08 \_ postgres: 19900, wal writer process   
gpadmin 3953970 251998 SNsl 0 19 00:00:00 \_ postgres: 19900, marketingpush dcw 172.20.79.51(44374) con677282 seg128 idle in transaction
gpadmin 3953970 260798 Ssl 19 0 00:00:00 \_ postgres: 19900, yunyingread dcw 172.20.79.51(61718) con677304 seg128 idle
gpadmin 3953970 260814 SNsl 0 19 00:00:00 \_ postgres: 19900, yunyingread dcw 172.20.79.51(62022) con677304 seg128 idle

我们使用ps命令查看相关的nice值和PRI值，发现父进程的nice值为0，而查询进程的nice值为19

## 2、参数设置
```
gpconfig -c gp_segworker_relative_priority -v 5 -m 5
```
当执行该命令的时候会出现如下异常：
>20190130:16:39:51:007505 gpconfig:kylin-203-130:gpadmin-[CRITICAL]:-GUC Validation Failed: gp_segworker_relative_priority cannot be changed under normal conditions. Please refer to gpconfig documentation.
GUC Validation Failed: gp_segworker_relative_priority cannot be changed under normal conditions. Please refer to gpconfig documentation.

这是由于在gp4.3.13.0版本以后，当我们去修改保护参数的时候会出现给异常，因此我们需要添加--skipvalidation，忽略校验
```
gpconfig -c gp_segworker_relative_priority -v 5 -m 5 --skipvalidation
```

## 3、重启生效
修改该参数需要重启gp服务器，否则不生效
```
gpstop -M fast -a
gpstart -a
```
## 4、修改回滚
当修改参数错误的时候导致集群无法启动，我们可以进入master-only模式，修改回之前的参数，然后重启gp集群，如果我们连master-only模式也无法进去，则需要修改master节点的postgresql.conf的配置，恢复默认配置，然后进入master-only模式，重新修改不正确的参数配置。
修改master节点的postgresql.conf配置文件，我们发现当我们执行gpconfig命令添加一个配置的时候会在postgresql.conf配置文件中添加相应的配置选项
```
gp_segworker_relative_priority=-20
```
我们发现该参数配置的值是无效的，范围在[0-20]之间，导致集群无法重启，master-only模式也无法进入，因此修改该参数值为默认值：20，然后进入master-only模式。
```
gpstart -m
```
然后执行gpconfig命令，修改为正确的参数配置，重启gp集群
```
gpconfig -c gp_segworker_relative_priority -v 5 -m 5 --skipvalidation
```
修改完成以后，执行重启命令
```
gpstop -r
```

## 5、查看配置
正确配置重启集群以后就能看到我们修改好的参数了。
```
gpconfig -s gp_segworker_relative_priority
```
>Values on all segments are consistent
GUC : gp_segworker_relative_priority
Master value: 5
Segment value: 5

# 六、GP集群nice方案制定
## 1、调整gpadmin用户默认优先级
凡是以gpadmin用户启动的进程，其nice值为-10，调高相关进程的优先级，从而在混部的时候与docker进程竞争资源能够更快的获取cpu时间片、获取更多的cpu时间片，优先保证GP进程稳定运行。
```
gpadmin soft priority -10
```
## 2、调整GP集群配置
通过调整gp_segworker_relative_priority参数，让查询进程的优先级低于主进程的优先级，从而保证主进程稳定运行，但是同时查询进程的优先级又高于docker等其他用户启动进程的优先级，其次保证查询进程的稳定性，以此来达到整个GP集群的nice调整。
```
gpconfig -c gp_segworker_relative_priority -v 5 -m 5 --skipvalidation
```




