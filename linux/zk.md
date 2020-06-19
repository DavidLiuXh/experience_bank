* 实际工作中用到Zookeeper集群的地方很多, 也碰到过各种各样的问题, 在这里作个收集整理, 后续会一直补充;
* 其中很多问题的原因, 解决方案都是google而来, 这里只是作次搬运工;
* 其实很多问题都跟配置有关, 只怪自己没好好读文档;
* 问题列表:
   **1. 一台 zk 节点重启后始终无法加入到集群中, 无法对外提供服务**
   **2. zk的log和snapshot占用大量空间**
   **3. 某台客户端上有的进程可以连接到zk, 有的无法连接**
   **4. 一台zk服务器无法对外提供服务,报错"Have smaller server identifier, so dropping 
the connection."**
   **5. zk客户端偶尔无法成功连接到zk server**
---
###### 一台 zk 节点重启后始终无法加入到集群中, 无法对外提供服务
- 现象: 使用zkCli.sh无法连接成功该zk节点
- 日志: 首先想到的是将该节点restart, 但问题依旧, 故查看zk的log, 有大量的如下日志
```
2017-07-18 17:31:12,015 - INFO  [WorkerReceiver Thread:FastLeaderElection@496] - Notification: 1 (n.leader), 77309411648 (n.zxid), 1 (n.round), LOOKING (n.state), 1 (n.sid), LOOKING (my state)
2017-07-18 17:31:12,016 - INFO  [WorkerReceiver Thread:FastLeaderElection@496] - Notification: 3 (n.leader), 73014444480 (n.zxid), 831 (n.round), LEADING (n.state), 3 (n.sid), LOOKING (my state)
2017-07-18 17:31:12,017 - INFO  [WorkerReceiver Thread:FastLeaderElection@496] - Notification: 3 (n.leader), 77309411648 (n.zxid), 832 (n.round), FOLLOWING (n.state), 2 (n.sid), LOOKING (my state)
2017-07-18 17:31:15,219 - INFO  [QuorumPeer:/0.0.0.0:2181:FastLeaderElection@697] - Notification time out: 6400
```
- 解决方案:
   1. Zookeeper本身的Bug: [FastLeaderElection - leader ignores the round information when joining a quorum](https://issues.apache.org/jira/browse/ZOOKEEPER-1514)
   2. 重启下当前的Leader, 产生新的Leader.
###### zk的log和snapshot占用大量空间
- 现象:  zk的datadir下的`version-2`下有大量的log和snapshot文件, 占用大量的磁盘空间
- 解决: 在配置文件里打开周期性自动清理的开关 `autopurge.purgeInterval=1`, 当然也可以通过 `autopurge.snapRetainCount`来设置需要保留的snapshot文件个数,默认是3;
###### 某台客户端上有的进程可以连接到zk, 有的无法连接
- 现象: 同一台客户端机器上启动多个相同的进程, 有些进程无法连接到zk集群
- zk服务端日志:
```
Too many connections from /x.x.x.x - max is x
```
- 解决: zk的配置中`maxClientCnxns`设置过小, 这个参数用来限制单个IP对zk集群的并发访问;
###### 一台zk服务器无法对外提供服务,报错"Have smaller server identifier, so dropping the connection."
* 现象:使用zkCli.sh无法连接成功该zk节点;
* 日志:  大量报错:`Have smaller server identifier, so dropping the connection.`
* 解决方案: 保持这台有问题zk的现状, 按myid从小到大依次重启其他的zk机器;
* 原因: zk是需要集群中所有机器两两建立连接的, 其中配置中的3555端口是用来进行选举时机器直接建立通讯的端口, 大id的server才会去连接小id的server，避免连接浪费.如果是最后重启myid最小的实例,该实例将不能加入到集群中, 因为不能和其他集群建立连接
###### zk客户端偶尔无法成功连接到zk server
* 现象: 同一台机器来运行的zk客户端, 偶发无法成功连接到zk server
* 分析: 
   1.  当时提供给业务一份sdk, sdk初始化时需要先连接zk, 初始化结束后断开zk的连接,业务将这份sdk用在了由fpm-php 处理的前端web请求的php代码中, 该业务的QPS在6K-8K左右, 相当于zk在处理大量的短连接请求;
   2. 在zk服务端监控下列命令的输出, overflowed和droped的数值在不断增加,说明 listen的accept queue有不断被打满的情况
```
[root@m1 ~]# netstat -s |grep -i listen
    53828 times the listen queue of a socket overflowed
    53828 SYNs to LISTEN sockets ignored
```
* 解决:
     1. 调整相关内核参数:/proc/sys/net/ipv4/tcp_max_syn_backlog和net.core.somaxconn
     2. zk服务端listen时的backlog用的是默认值50, zk没参数用来设置这个,有这个issue:[Configurable listen socket backlog for the client port](https://issues.apache.org/jira/browse/ZOOKEEPER-974), 里面提供了patch;
   3. 避免客户端有大量短连接的方式连接zk服务;
* 深究:
   关于tcp连接队列,这篇文章很不错: [How TCP backlog works in Linux](http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html)