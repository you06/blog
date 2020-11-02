title: TiUP 使用指南
author: you06
date: 2020-11-02 21:24:32
tags:
---
`TiUP`是一个`TiDB`的组件管理工具，我本人的使用体验是（在会用的情况下）非常好用，可惜有的时候不会用，这里记载一些自己踩过的坑以及常用的使用方法，意在弥补文档不全面的情况。

## TiDB Playground

回想起当时做小作业的时候，为了起一个`TiDB`集群，不管是用`TiDB`的`Ansible`还是`Docker Compose`都对我这个萌新（和低配 Mac）非常不友好，直到`TiUP`问世。

从`TiUP Playground`最基本的使用方式说起，不加任何参数启动`Playground`，会看到以下结果。

```sh
» tiup playground
...
Start tiflash instance
...
CLUSTER START SUCCESSFULLY, Enjoy it ^-^
To connect TiDB: mysql --host 127.0.0.1 --port 37257 -u root
To view the dashboard: http://127.0.0.1:2379/dashboard
To view the Prometheus: http://127.0.0.1:9090
To view the Grafana: http://127.0.0.1:3000
```

其实我起`Playground`大部分时候只是想做一些功能性测试，并不希望看到`Prometheus`和`Grafana`以及`tiflash`，所以我常用的命令是：

```sh
tiup playground --tiflash=0 --monitor=false
```

在使用`Ctrl + C`发送`SIGINT`时，会看到`TiUP`进程卡在了这样一个状态，正在等待某某某进程退出，然后下一个，到全部退出可能要花费十多秒的时间，主要是等待`TiDB`退出的时间特别长，以及`TiUP`会删除掉`TiDB`集群所生成的文件。使用`Ctrl + C`发送第二个`SIGINT`信号会让`Playground`和它所拉起来的进程强制退出（`SIGKILL`）。

```sh
^CPlayground receive signal:  interrupt
Got signal interrupt (Component: playground ; PID: 6355)
Wait pd(6377) to quit...
pd quit
Wait tikv(6383) to quit...
tikv quit
Wait tidb(6385) to quit...
tidb quit
```

## TiDB Cluster

如果要起一个生产环境用的集群，需要使用`TiDB Cluster`这个组件，当然我不是专业的DBA，也没有在生产环境实践过，这里所说的，是一个程序员启动一个开发用集群的实践。主要流程请参考[官方文档](https://docs.pingcap.com/zh/tidb/stable/tiup-cluster)，本文只写自己遇到的坑。

### 1. 编辑拓扑文件

最初写`TiUP`的拓扑文件，因为找不到文档，所以对着代码里的结构体定义在写，写的十分痛苦，建议直接抄 example： https://github.com/pingcap/tiup/blob/master/examples/topology.example.yaml 。

### 2. 部署启动集群

认真看官方文档的小伙伴，应该不用来这里抄命令了。

```sh
» tiup cluster deploy cluster-name nightly topology.yaml
» tiup cluster list
Name          User  Version  Path                                                      PrivateKey
----          ----  -------  ----                                                      ----------
amend-down    tidb  v4.0.7   /home/tidb/.tiup/storage/cluster/clusters/cluster-name    ...
» tiup cluster start cluster-name
```

### 3. patch

有的时候想要测试自己编译的组件，`TiUP`提供了`patch`命令，但是大家非常容易翻一个常见错误，以`TiDB`为例：

```sh
» tiup cluster patch my-cluster tidb-server -R tidb
...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.2.0/tiup-cluster patch my-cluster tidb-server -R tidb

Error: exit status 2

Verbose debug logs has been written to /home/tidb/tiup/logs/tiup-cluster-debug-2077-11-04-05-14-00.log.
Error: run `/home/tidb/.tiup/components/cluster/v1.2.0/tiup-cluster` (wd:/home/you06/.tiup/data/SFC8xCY) failed: exit status 1
```

因为根据 DBA 的使用习惯，一般想要直接使用下载下来的 tar 包，所以 patch 命令所指定的文件也需要是一个 tar 包（快兼容一下啊(╯‵□′)╯︵┻━┻）。

```sh
tar xzf tidb-server.tar.gz tidb-server
tiup cluster patch my-cluster tidb-server -R tidb
```
