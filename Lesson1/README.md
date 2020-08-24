# Lesson1-编译部署 TiDB

本文是 [High Performance TiDB 课程系列](https://docs.qq.com/sheet/DSlBwS3VCb01kTnZw?tab=BB08J2) Lesson 1 的课程作业。

## 题目描述

本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：

- 1 TiDB
- 1 PD
- 3 TiKV

改写后：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的日志。

**输出：一篇文章介绍以上过程**

## TiUP 一键安装

[TiDB 数据库快速上手指南](https://docs.pingcap.com/zh/tidb/stable/quick-start-with-tidb)中介绍了如何快速上手体验 TiDB 分布式数据库。最简单的方式就是装完 `TiUP` 以后直接 `tiup playground` 就启动了最新版本的 TiDB 集群，其中 TiDB、TiKV 和 PD 实例各 1 个。

也可以指定实例个数 `tiup playground --db 1 --pd 3 --kv 3` 。在启动完成以后，可以通过 mysql 客户端连接，也可以打开 dashboard 可视化地查看集群情况。

```bash
CLUSTER START SUCCESSFULLY, Enjoy it ^-^
To connect TiDB: mysql --host 127.0.0.1 --port 4000 -u root
To view the dashboard: http://127.0.0.1:2379/dashboard
To view the Prometheus: http://127.0.0.1:9090
To view the Grafana: http://127.0.0.1:3000
```

## Build from source

再试试完整从源码编译吧，毕竟还要修改源码。

在此之前 go 和 Rust 我都已经装过了，可以直接进入正题。 

```bash
cd ~
mkdir tidb
cd tidb
git clone https://github.com/tikv/tikv.git
git clone https://github.com/pingcap/pd.git
git clone https://github.com/pingcap/tidb.git

cd pd
make

cd ../tidb
make

cd ../tikv
cargo build
```

毫无障碍地编译成功。启动的话虽然可以一个个打开二进制文件，但是还得输长长的参数。我发现 `tiup` 可以手动指定二进制文件启动，于是我们又可以傻瓜式一键部署了~

```bash
cd ~/tidb
tiup playground \
	--db 1 --db.binpath ./tidb/bin/tidb-server \
	--pd 1 --pd.binpath ./pd/bin/pd-server \
	--kv 3 --kv.binpath ./tikv/target/debug/tikv-server
```

## 修改源码

### SQL 执行原理

[TiDB 源码阅读系列文章（三）SQL 的一生](https://pingcap.com/blog-cn/tidb-source-code-reading-3/)中讲解了 SQL 的执行过程。

1. 当 TiDB 和客户端的连接建立好之后，等待发来的包，并处理。`server/conn.go` 中的 `clientConn.Run()` 读取包，然后调用 `dispatch()`，根据 Command 的类型分发处理。这部分是 MySQL 协议层的范围，最常用的 Command 是 `COM_QUERY` ，大多数 SQL 语句都是它。
2. 接着进入 `session.Execute`，SQL 核心层。
3. session 经过 parse、plan 的前戏后，生成 executor ，最后就是调用 `Next`（火山模型），开始最终的执行了~

![SQL 层执行过程](https://download.pingcap.com/images/blog-cn/tidb-source-code-reading-3/2.png)

### 定位源码位置

所以事务的开始应该是 executor 执行的地方，我们可以从 parser 的语法树里关于事务的语句以及从执行事务的 executor 中定位到事务开始的地方。

在 `github.com/pingcap/parser/ast` 里找到开始事务的语句类型 `BeginStmt` ，find usage 可以找到 `executor/simple.go` 里面的函数 `executeBegin` ，这应该就是我们想要的事务开始的 executor 了。在函数的最前面加上一句 `logutil.Logger(ctx).Info("hello transaction")` ，重新编译并运行。

### 看日志

在 `~/.tiup/data` 下有个奇奇怪怪的名字 `S7pQVK1` ，应该是临时生成的集群名。`S7pQVK1/tidb-0/tidb.log` 就是 tidb-server 的日志了。另外一种看日志的方式是在 TiDB Dashboard 里的日志搜索，非常方便。

打开日志惊讶地发现里面已经有很多 `[INFO] [simple.go:555] ["hello transaction"]` 了，并且会一直不断地生成。不知道是什么后台任务会一直启动事务。不过自己输入 `BEGIN` 开启事务，会输出 `["hello transaction"] [conn=1]` ，多了一个连接号。

至此，TiDB 初上手就完成了~😊