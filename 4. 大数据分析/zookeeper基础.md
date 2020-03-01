# ZooKeeper

## 1. ZooKeeper概述

### 1.1 概述

ZooKeeper是一个分布式协调服务的开源框架，本质上是一个分布式的小文件存储系统。

### 1.2 特性

* 全局一致性：每台机器的数据目录是相同的
* 可靠性：消息被其中一台服务器接收，即被所有机器接收
* 顺序性：包括全序和偏序
* 数据更新原子性：一次数据更新要么成功（半数以上节点成功），要么失败，不存在中间状态
* 实时性：保护客户端在一段时间间隔范围内获得服务器的更新或失效信息，但由于网络延迟原因，不能保证2个客户端同时被接收

### 1.3 集群角色

**Leader**

集群工作的核心，事物请求（写操作）的唯一调度和处理者，保证集群事务处理的顺序性，对于增删改这样的事务则需要统一转发给leader处理

**Follower**

处理客户端非事务请求（读操作），转发请求给leader，选举投票Leader

**Observer**

观察集群的最新状态变化并将这些状态同步，不会参与任何形式的投票，只提供非事务服务

### 1.4 集群搭建

需要java环境（jdk），安装 leader+follower 大致过程如下：

* 配置主机名称到IP地址映射配置
* 修改ZooKeeper配置文件
* 远程复制分发安装操作
* 设置myid
* 启动ZooKeeper集群

若想用 Observer 模式，可在对应节点的配置文件添加如下配置

```shell
peerType=observer
```

其次，必须在配置文件指定哪些节点被指定为 Observer，如：

```shell
server.1:localhost:2181:3181:observer
```



## 2. Zookeeper数据模型

采用树形层次结构，每个节点被称为一个Znode

1. Znode兼具文件和目录两种特点，
2. Znode具有原子性操作
3. Znode存储数据大小有限制，每个Znode至多1M
4. Znode通过路径引用，路径必须是绝对路径

### 2.1 数据结构图

每个Znode由3部分组成：

1. stat：此为状态信息，描述该Znode的版本，权限等信息
2. data：与该Znode关联的数据
3. children：该Znode下的子节点

### 2.2. 节点类型

Znode有两种，分别为临时节点和永久节点，节点的类型在创建时即被指定，不能更改

* **临时节点：**客户端断开连接，节点自动删除，不允许拥有子节点

* **永久节点：**只有客户端显示执行删除操作的时候，他们才能被删除

具有序列化的特性，若创建时指定的话，则该Znode的名字后面会自动追加一个不断增加的序列号，该序列号对于父节点唯一，这样便于记录每个子节点创建的先后顺序，格式为“%10d"

* `PERSISTENT`：永久节点
* `EPHEMERAL`：临时节点
* `PERSISTENT_SEQUERNTIAL`：永久节点、序列化
* `EPHEMERAL_SEQUENTIAL`：临时节点、序列化

### 2.3 节点属性

每个znode都包含了一系列的属性，通过命令get，可以获得节点的属性

`dataVersion`：数据版本号，每次对节点进行set操作，dataVersion的值都会增加1（即使设置的是相同的数据）

`cversion`：子节点版本号，当znode的子节点有变化时，cversion的值就会增加1

`aclVersion`：ACL的版本号

`cZxid`：Znode创建的事务id

`mZxid`：Znode被修改的事务id，即每次对znode的修改都会更新mZxid。

`ctime`：节点创建时的时间戳

`mtime`：节点最新一次更新发生时的时间戳

`ephemera10wner`：该值若不是0，则表示的是临时节点绑定的session id



## 3. ZooKeeper shell

### 3.1 客户端连接

运行 `zkCli.sh -server ip` 进入命令行工具

输入help，输出zk shell 提示：

### 3.2 shell操作

**创建节点**

```shell
create [s] [-e] path data acl
```

其中，-s 或 -e 分别指定节点特性，顺序或临时节点，若不指定，则表示持久节点；acl用来进行权限控制

**读取节点**

与读取相关的命令有 ls 命令和 get 命令，ls 命令可以列出 Zookeeper指定节点下的所有子节点，只能查看指定节点下的第一级的所有子节点，get 命令可以获取 Zookeeper指定节点的数据内容和属性信息。

```shell
ls path [watch]
get path [watch]
ls2 path [watch]
```

**更新节点**

```shell
set path data [version]
```

data就是要更新的新内容，version表示数据版本，即dataVersion

**删除节点**

```shell
delete path [version]
```

若删除节点存在子节点，那么无法删除该节点，必须先删除子节点，再删除父节点

```shell
Rmr path
```

可以递归删除节点

**quota**

该命令是对节点增加限制

````shell
setquota -n|-b val path
````

* n：表示子节点的最大个数，软限制
* b：表示数据值的最大长度
* val：子节点最大个数值或数据值的最大长度值
* path：节点路径

列出指定节点的quota，数据长度为-1则表示没限制

```shell
listquota path
```

删除quota的命令

```shell
delquota [-n|-b] path	
```

**其它命令**

history：列出历史命令

redo：该命令可以重新执行指定命令编号的历史命令，命令编号可以通过 history 查看

## 4. ZooKeeper Watcher

ZooKeeper提供了**分布式数据发布/订阅功能**，一个典型的发布/订阅模型系统统一定义了一种一对多的订阅关系，能让多个订阅者同时监听某一个主题对象，当这个主题对象自身状态发生变化时，会通知所有订阅者，使它们能够作出相应的处理。

ZooKeeper中，引入了Watcher机制来实现这种分布式的通知功能。ZooKeeper允许客户端向服务端注册一个Watcher监听，当服务端的一些事件触发了这个Watcher，那么就会向指定的客户端发送一个事件通知来实现分布式的通知功能。

触发事件种类很多，如：节点的增删改等

Watcher可以概括为以下三个过程：客户端向服务端注册Watcher、服务端事件发生触发Watcher、客户端回调Watcher得到触发事件情况

### 4.1 Watcher机制特点

**一次性触发**

事件发生触发监听，一个watcher event就会被发送到设置监听的客户端，这种效果是一次性的，后续再发生同样的事件，不会再次触发。

**事件封装**

ZooKeeper使用WatchEvent对象来封装服务端事件并传递

WatchEvent包含了每一个事件的三个基本属性：

<font color='green'>通知状态（keeperState），事件类型（EventTpye）和节点路径（path）</font>

**event异步发送**

Watcher的通知事件从服务端发送到客户端是异步的。

**先注册再触发**

Zookeeper中的watch机制，必须是客户端先去服务端注册监听，这样事件发送才会触发监听，通知给客户端。

### 4.2 通知状态和事件类型

同一个事件类型在不同的通知状态中代表的含义有所不同，下表列举了常见的通知状态和事件类型。

| KeeperState        | EventType        | 触发条件                                      | 说明                                                         |
| ------------------ | ---------------- | --------------------------------------------- | ------------------------------------------------------------ |
|                    | None（-1）       | 客户端与服务端成功建立连接                    |                                                              |
| SyncConnected（0） | NodeCreated      | Watcher监听的对应数据节点被创建               |                                                              |
|                    | NodeDeleted      | Watcher监听的对应数据节点被删除               | 此时客户端和服务器处于连接状态                               |
|                    | NodeDataChanged  | Watcher监听的对应数据节点的内容发生更改       |                                                              |
|                    | NodeChildChanged | Watcher监听的对应数据节点的子节点列表发生更改 |                                                              |
| Disconnected       | None(-1)         | 客户端与Zookeeper服务器断开连接               | 此时客户端和服务器处于断开状态                               |
| Expired（-112）    | Node(-1)         | 会话超时                                      | 此时客户端会话失效，通常同时也会收到SessionExpiredException异常 |

其中，<font color='red'>连接状态事件（type=None, path=null）不需要客户端注册，</font>客户端只要有需要直接处理就行了。

### 4.3 Shell客户端设置 watcher

设置节点数据变动监听

```shell
get path watch	
```

通知另外一个客户端更改节点数据：

```shell
set path data
```

此时设置监听的节点会收到通知

## 5. ZooKeeper Java API

**`org.apache.zookeeper.Zookeeper`**

Zookeeper是在 Java 中客户端主类，负责建立与 zookeeper 集群的会话，并提供方法进行操作。

**`org.apache.zookeeper.Watcher`**

Watcher接口表示一个标准的事件处理器，其定义了事件通知相关的逻辑，包含 KeeperState 和 EventType 两个枚举类，分别代表通知状态和事件类型，同时定义了事件的回调方法：process（WatchedEvent event）。

process 方法是Watcher接口中的一个回调方法，当 ZooKeeper 向客户端发送一个 Watcher 事件通知时，客户端就会对相应的 process 方法进行回调，从而实现对事件的处理。

### 5.1 基本使用

建立 java maven项目，引入maven pom坐标

```java
<dependency>
     <groupId>org.apache.zookeeper</groupId>
     <artifactId>zookeeper</zrtifactId>
     <version>3.4.9</version>
</dependency>
```

```java
public static void main(String[] args) throws Exception{
    ZooKeeper zk =new ZooKeeper("node-1:2181,node-2:2181",30000,new Watcher(){
        @Override
        //这里是事件通知的回调方法
        public void process(WatchedEvent event){
            System.out.println(event.getType());
            System.out.println(event.getPath());
            System.out.println(event.getState());
        }
    });
    zk.create("/myGirls","性感的",getBytes("UTF-8"),
              ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT_SEQUENTIAL);
    zk.close();
}
```

### 5.2 更多操作示例

续上述12行代码后面

```java
// 相当于对节点，设置了数据变化的监听，一旦节点数据改变，监听就会触发
zk,getData("/myGirls",true，null)
// 这里对节点数据进行了修改，那么设置的监听就会触发
zk.setData("/myGirls","美丽的".getBytes(),-1);
```

## 6. ZooKeeper选举机制

zookeeper默认的算法是FastLeaderElection，采用投票数大于半数则胜出的逻辑

### 6.1 概念

**服务器ID**

* 比如有三台服务器，编号分别是1，2，3

* 编号越大在选择算法中的权重越大

**选举状态**

* LOOKING，竞选状态

* FOLLOWING： 随从状态，同步leader状态，参与投票

* OBSERVING：观察状态，同步leader状态，不参与投票

* LEADING：领导者状态

**数据ID**

* 服务器中存放的最新数据version
* 值越大说明数据越新，在选举算法中数据越新，权重越大

**逻辑时钟**

也叫投票的次数，同一轮投票过程中的逻辑时钟值是相同的，每投完一次票这个数据就会增加，然后与接收到的其它服务器返回的投票信息中的数值相比，根据不同的值作出不同的判断。

### 6.2 全新集群选举

假如目前有3台服务器，每台服务器均没有数据，它们的编号分别是1,2,3，按编号依次启动，选举过程如下：

* 服务器1启动，给自己投票，然后发投票信息，由于其它机器没有启动，因此一直处于Looking状态
* 服务器2启动，给自己投票，同时与服务器1交换结果，由于服务器2编号大，胜出，此时投票数大于服务器数的一半，因此服务器2成为领导者，服务器1成为小弟
* 服务器3启动，给自己投票，同时与服务器1,2交换结果，尽管服务器3编号大， 但由于投票结束，因此只能成为小弟

### 6.3 非全新集群选举

对于运行正常的zookeeper·集群，中途有机器down掉，需要重新选举

* 数据ID：数据新的version就大，数据每次更新都会更新version
* 服务器ID：就是我们配置的 myid中的值，每个机器一个
* 逻辑时钟：这个值从0开始递增，每次选举对应一个值。如果在同一次选举中，这个值是一致的。

因此选举标准变为：

1. 逻辑时钟小的选举结果被忽略，重新投票
2. 统一逻辑时钟后，数据id大的胜出
3. 数据id相同的情况下，服务器id大的胜出

根据这个规则，选出leader



## 7. ZooKeeper典型应用

### 7.1 数据发布与订阅（配置中心）

发布与订阅模型即发布者将数据发布到ZK节点上，供订阅者动态获取数据，实现配置信息的集中式管理和动态更新。应用在启动时会主动获取一次配置，同时，在节点上注册一个Watcher，这样一来，以后每次配置有更新的时候，都会实时通知到订阅的客户端，从而达到获取最新配置信息的目的。

### 7.2 命名服务（Naming Service）

在分布式系统中，<font color='red'>通过使用命名服务，客户端应用能够根据指定名字来获取资源或服务的地址，提供者等信息</font>，被命名的实体通常可以是集群中的机器，提供的服务地址，远程对象等等——这些我们都可以称之为名字，其中，较为常见的就是一些分布式服务框架中的服务地址列表，通过调用ZK提供的创建节点的 API，能够很容易创建一<font color='red'>个全局唯一的 path</font>，这个 path 就可以作为一个名称。

阿里开源的分布式服务框架 Dubbo 中使用 ZooKeeper 来作为其命名服务，维护全局的服务地址列表

### 7.3 分布式锁

得益于ZooKeeper要保证数据的强一致性，锁服务分为两类，一个是保持独占，另一个是控制时序

**保持独占**

所谓保持独占，就是所有试图来获取这个锁的客户端，最终只有一个可以成功获得这把锁，通常的做法是把zk上的一个 znode 看作是一把锁，通过`create znode` 的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁。

**控制时序**

控制时序，就是所有试图来获取这个锁的客户端，最终都会被安排执行，只是有个全局时序。









**参考资料**

1.[如果有人问你ZooKeeper是什么，就把这篇文章发给他。](https://juejin.im/post/5baf7db75188255c3d11622e)

2.[Zookeeper集群搭建以及python操作zk](https://www.cnblogs.com/xiao987334176/p/10103619.html#autoid-1-5-0)

3.[大数据ZooKeeper快速入门 ](https://edu.aliyun.com/course/1343?spm=5176.10731542.0.0.35619148ZZwZVT)