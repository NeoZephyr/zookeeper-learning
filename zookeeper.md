zoo.cfg
```sh
# 存放日志文件、快照文件
dataDir=/data/zookeeper
clientPort=2181
```
```sh
zkServer.sh start
```
```sh
zkCli.sh

ls -R /
create /app1
create /app2

# 创建锁
create -e /lock

# 监控
stat -w /lock
```

```sh
create -e /master "m1:6969"
create -e /master "m2:6969"
stat -w /master
```
```sh
ls -w /workers
create -e /workers/w1 "w1:9696"
```

zookeeper 数据一致性
1. 先到达 leader 的写请求先处理
2. 来自给定客户端的请求按照发送顺序执行

集群配置
```sh
dataDir=/data/zk/quorum/node1
clientPort=2181

# 前一个端口用于 quorum 通信，后一个端口用于 leader 选举端口
server.1=127.0.0.1:3330:3333
server.2=127.0.0.1:4440:4444
server.3=127.0.0.1:5550:5555
```
在为每个 zookeeper 节点的 data 目录下创建 myid 文件，内容分别为 1, 2, 3

```sh
zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```

条件更新
```sh
# -v 匹配版本
set -s -v 0 /count 1
set -s -v 1 /count 2

# -w 监听
get -s -w /count
```

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);

CuratorFramework client = CuratorFrameworkFactory.newClient(connectString, retryPolicy);
CuratorFramework client = CuratorFrameworkFactory.buidler().connectString(connectString).retryPolicy(retryPolicy).build();

client.start();
```
```java
// 同步版本
client.create().withMode(CreateMode.PERSISTENT).forPath(path, data);

// 异步版本
client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);

// 使用 watch
client.getData().watched().forPath(path);
```

```java
private CuratorFramework client;
private String connectString = "localhost:2181";
private RetryPolicy retryPolicy;

public void setUp() {
  retryPolicy = new ExponentialBackoffRetry(1000, 3);
  client = CuratorFrameworkFactory.newClient(connectString, retryPolicy);
  client.start();
}

public void tearDown() {
  client.close()
}

public void testSync() {
  String path = "sync";
  byte[] data = {'1'};
  client.create().withMode(CreateMode.PERSISTENT).forPath(path, data);
  byte[] actualData = client.getData().forPath(path);
  client.delete().forPath(path);
  client.close();
}

public void testAsync() {
  String path = "async";
  byte[] data = {'2'};
  CountDownLatch latch = new CountDownLatch(1);

  client.getCuratorListenable().addListener((CuratorFramework c, CuratorEvent event) -> {
    switch (event.getType) {
      case CREATE:
        // print event.getPath
        c.getData().inBackground().forPath(event.getPath());
        break;
      case GET_DATA:
        c.delete().inBackground().forPath(path);
        break;
      case DELETE:
        latch.countDown();
        break;
    }
  });

  client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);
  latch.await();
  client.close();
}

public void testWatch() {
  String path = "watch";
  byte[] data = {'3'};
  byte[] newData = {'4'};
  ClientDownLatch latch = new CountDownLatch(1);

  client.getCuratorListenable().addListener((CuratorFramework c, CuratorEvent event) -> {
    switch (event.getType()) {
      case WATCHED:
        WatchedEvent we = event.getWatchedEvent();

        if (we.getType() == Watcher.Event.EventType.NodeDataChanged && we.getPath().equals(path)) {
          byte[] actualData = c.getData().forPath(path);
        }

        latch.countDown();
        break;
    }
  });

  client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);
  byte[] actualData = client.getData().watched().forPath(path);
  client.setData().forPath(path, newData);
  latch.await();
  c.delete().forPath(path);
}

public void testWatchAndCallback() {
  String path = "/callback";
  byte[] data = {'5'};
  byte[] newData = {'6'};
  CountDownLatch latch = new CountDownLatch(2);

  client.getCuratorListenable().addListener((CuratorFramework c, CuratorEvent event) -> {
    switch (event.getType()) {
      case CREATE:
        client.getData().watched().forPath(path);
        client.setData().forPath(path, newData);
        latch.countDown();
        break;
      case WATCHED:
        WatchedEvent we = event.getWatchedEvent();

        if (we.getType() == Watcher.Event.EventType.NodeDataChanged && we.getPath().equals(path)) {
          byte[] actualData = c.getData().forPath(path);
        }

        latch.countDown();
    }
  });

  client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);
  latch.await();
  client.delete().forPath(path);
}
```
zookeeper 配置
```
# 保存快照文件的目录
dataDir
# 事务日志文件目录
dataLogDir
```

```
echo ruok | ncat localhost 2181
echo conf | ncat localhost 2181
echo stat | ncat localhost 2181

# 临时节点信息
echo dump | ncat localhost 2181

# 查看 watch
echo wchc | ncat localhost 2181

echo srvr | ncat localhost 2181
```

客户端写请求 -> observer/follower -> forward 到 leader -> propose 到 follower -> follower accept -> leader 发送 commit


observer 不参与提交和选举的投票过程，只会接受 leader 节点的 inform 消息。可以通过向集群中添加 observer 节点提高集群读性能
observer -> 跨数据中心部署

配置 observer
```
server.4=127.0.0.1:5550:5555:observer
```

动态配置
```java
String digest = DigestAuthenticationProvider.generateDigest("super:jingguo");
```
```
export SERVER_JVMFLAGS
export SERVER_JVMFLAGS=-Dzookeeper.DigestAuthenticationProvider.superDigest=<digest>
```
动态配置的配置文件
```
reconfigEnabled=true
dataDir=/data/zookeeper
syncLimit=5
tickTime=2000
initLimit=10
dynamicConfigFile=conf/dyn.cfg
```
```
server.1=ip1:2222:2223:participant;ip1:2181
server.2=ip2:2222:2223:participant;ip2:2181
server.3=ip3:2222:2223:participant;ip3:2181
```
```
zkServer.sh start zoo_reconf.cfg
```
```
zkCli.sh -server ip1:2181
addauth digest super:jingguo
config
reconfig -remvoe 3
config
```


```sh
zkTxnLogToolkit.sh log.1
```