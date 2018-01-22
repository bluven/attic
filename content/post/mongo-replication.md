---
title: "MongoDB复制集实践"
date: 2018-01-20T21:34:45+08:00
---

1. 安装MongoDB
-------------

先去官网[http://www.mongodb.org/downloads]([http://www.mongodb.org/downloads)下载实验平台的安装包，我的系统是64位的centOS，下载的版本是mongodb-linux-x86_64-2.6.7.tgz。 

下载结束后，解压：`tar -xvzf mongodb-linux-x86_64-2.6.7.tgz`，将解压后的MongoDB放到`/opt/mongo`下，同时创建MongoDB在linux下的默认数据存储目录：`mkdir -p /data/db`。 

修改PATH，加入MongoDB的bin路径，不嫌麻烦的人也可以直接用全路径调用MongoDB的各种命令。

启动MongoDB是很容易的，直接输入`mongod`就可以了，或者全路径调用`./opt/mongo/bin/mongod`。
启动MongoDB还可以指定配置文件和数据保存目录，因为是实验，我这里就不使用其他方式了。

2. 复制集介绍（Setup ReplicaSet）
---------------

先看一下复制的介绍：
    
> Replication is a way of keeping identical copies of your data on multiple servers and is recommended for all production deployments. Replication keeps your application running and your data safe, even if something happens to one or more of your servers.

简单说来就是在多台mongo服务器上保存同样的数据，这样可以让生产环境下的系统更健壮，同时保证了数据安全。

我们可以通过创建一个复制集(replica set)的方式来实现MongoDB的复制。 一个复制集是一组拥有一个主服务器(primary server)及多个辅助服务器(secondary server)，其中辅助服务器用来复制主服务器的数据，如果主服务器宕机了，辅助服务器就可以选出一个新的主服务器来继续提供服务。

3. 一分钟的测试配置
-------------------

启动一个没有连接数据库的mongo shell：`mongo --nodb`。

在shell中输入如下命令：

`> replicaSet = new ReplSetTest({"nodes" : 3})`

这条命令创建了具有一个具有3个服务器的复制集：一个主+2个辅助。

输入如下命令来启动这些服务器：
`> replicaSet.startSet(); replicaSet.initiate()`

startSet负责启动mongod进程，initiate负责配置复制集。这两条命令可以单独输入，但是因为在startSet执行后，shell会不停地输出一些消息，所以放在一起执行是最好的。

有一点需要注意的是，startSet执行后可能会报告如下错误： 

    replSet can't get local.system.replset config from self or any seed (EMPTYCONFIG)。
    
这是因为复制集的3个服务器尝试着用`hostname:port`（我的情况是：bluven-centos:31000）来访问其他服务器，而hostname无法识别的原因。要解决这个问题，可以修改/etc/hosts文件，将127.0.0.1映射到你的hostname上。要查询自己的hostname，请输入`uname -a`，第二个数据就是你的hostname。比如我的虚拟机得到的数据是：
    
    Linux bluven-centos 2.6.32-358.el6.x86_64 #1 SMP Fri Feb 22 00:31:26 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux
    
其中bluven-centos就是我的hostname。

除了需改hosts文件外，还可以在initiate命令中传入参数来保证网络正确性，具体操作请看文档：[http://docs.mongodb.org/manual/tutorial/change-hostnames-in-a-replica-set/](http://docs.mongodb.org/manual/tutorial/change-hostnames-in-a-replica-set/)。

如果网络访问没有问题，这个集群已经可以使用了。此时集群会有3个服务器进程，分别占用了端口：31000, 31001, and 31002。在`/data/db/`目录下应该有3个服务器的数据目录：testReplSet-0  testReplSet-1  testReplSet-2。


4. Let's Play!
----------------

用`mongo --nodb`另起一个mongo shell，在shell中输入如下命令：

    > conn1 = new Mongo("localhost:31000")
    connection to localhost:31000
    > db = conn1.getDB('test')
    test
    testReplSet:PRIMARY>

注意：当获取db的时候，提示符由“>”换成了“testReplSet:PRIMARY”，表示你当前连接的是primary服务器。


执行如下命令`db.isMaster()`, 会得到如下信息：

    {
        "setName" : "testReplSet",
        "setVersion" : 1,
        "ismaster" : true,
        "secondary" : false,
        "hosts" : [
            "bluven-centos:31000",
            "bluven-centos:31002",
            "bluven-centos:31001"
        ],
        "primary" : "bluven-centos:31000",
        "me" : "bluven-centos:31000",
        "maxBsonObjectSize" : 16777216,
        "maxMessageSizeBytes" : 48000000,
        "maxWriteBatchSize" : 1000,
        "localTime" : ISODate("2015-01-26T02:29:58.776Z"),
        "maxWireVersion" : 2,
        "minWireVersion" : 0,
        "ok" : 1
    }
    
因为我们使用ReplSetTest创建的复制集，所以默认的名字是"testReplSet"。可以从"isMaster"的值看出当前的数据库确实是主服务器。

注：isMaster这个名字来源于MongoDB早期的Master-Slave模式，现在这种模式已经被放弃，推荐使用ReplicaSet了，但是这个方法及属性名保留了下来。
  
执行如下命令创建一些数据：
 
      for (i=0; i<1000; i++) { 
          primaryDB.coll.insert({count: i}) 
      }
  
然后另起一个mongo shell，还是用命令`mongo --nodb`,在shell中执行如下命令：
  
      > conn2 = new Mongo("localhost:31001")
      > db = conn2.getDB("test")
      
这次我们连接到了辅助服务器，默认情况下辅助服务器是不能接受读写请求的，如果尝试执行读操作，如下：
    
    > db.coll.find()
    error: { "$err" : "not master and slaveok=false", "code" : 13435 }

这是为了防止客户端不小心读到不合适的数据，如果客户端想要从辅助服务器读取数据，必须声明：

    > conn2.setSlaveOk()
    > secondaryDB.coll.find()
    { "_id" : ObjectId("5037cac65f3257931833902b"), "count" : 0 } 
    { "_id" : ObjectId("5037cac65f3257931833902c"), "count" : 1 } 
    { "_id" : ObjectId("5037cac65f3257931833902d"), "count" : 2 } ...
    { "_id" : ObjectId("5037cac65f3257931833903c"), "count" : 17 } 
    { "_id" : ObjectId("5037cac65f3257931833903d"), "count" : 18 } 
    { "_id" : ObjectId("5037cac65f3257931833903e"), "count" : 19 }

现在试着往辅助服务器写入数据：
    
    > secondaryDB.coll.insert({"count" : 1001})
    WriteResult({ "writeError" : { "code" : undefined, "errmsg" : "not master" } })
    
辅助服务器唯一能接受的写操作是从主服务器复制数据，尝试在客户端写入数据是不会成功的。

下面让我么尝试一下MongoDB的自动灾难恢复(automatic failover), 在连接主服务器的mongo shell中执行如下命令关闭主服务器：
    
    > db.adminCommand({"shutdown" : 1})
    
稍等片刻， 然后在连接辅助服务器的mongo shell中执行如下命令：

    > db.isMaster()

如果出现的结果没有显示这个服务器成为了主服务器，那就是另一个服务器成为了主服务器，读者可以自行创建新的连接测试。

哪个辅助服务器能成为主节点的影响因素很多，一般情况下，首先发现主服务器不能连接的辅助服务器会优先成为主服务器。

目前为止，我们已经实验了复制集的读写，复制，灾难恢复工等功能，是时候把他们放到生产环境下了。在进入下一环节之前，先把这个复制集关掉，在开启复制集的那个mongo shell中执行如下命令：

    replicaSet.stopSet()
    
这个shell肯能在不停地输出信息，只管输入命令就行，输出不影响输入的，前提是一定要输对了。

5. 多机实战
--------------

每个复制集都得有个名字，我的复制集的名字是：spock。我的实验环境是3台CentOS虚拟机，IP分别是：192.168.1.31，192.168.1.32，192.168.1.33，为了后面描述方便，分别叫他们spock-1, spock-2,spock-3，其中spock-1是主服务器。

现在，在3台机器上分别启动mongod进程，这次与test环境不同，我们需要在mongod启动的时候指定其复制集名字，输入如下命令：
    
    mongod --replSet spock

现在，3个mongod进程已经启动，他们也有了公共的集合名，但他们还不知道彼此的存在，需要进一步的配置。

启动一个mongo shell：`mongo --nodb`。

在shell中先创建复制集节点配置数据：
    
    > config = {
        "_id" : "spock",
        "members" : [
            {"_id" : 0, "host" : "192.168.1.31:27017"},
            {"_id" : 1, "host" : "192.168.1.32:27017"},
            {"_id" : 2, "host" : "192.168.1.33:27017"}
    ]}

在shell中建立到spock-1的链接：

    > // connect to server-1
    > db = (new Mongo("192.168.1.31:27017")).getDB("test") >
    > // initiate replica set
    > rs.initiate(config)
    {"info" : "Config now saved locally.  Should come online in about a minute.",
    "ok" : 1 }

spock-1会解析这个配置，并将配置数据发送给其他两个节点，一旦3个节点都加载了这个配置，他们就会选出一个主服务器。在选举主服务器期间，该复制集不能接受读写请求，选举结束后才可以接受。

注1：如果你创建的复制集是全新的，那么你可以将配置数据传送给任意节点来初始化该复制集；如果其中一个节点有数据，而你又需要其他节点复制该数据，那么就将配置数据传送给该节点，否则请保证所有的节点数据是空的。

注2：如果你想把一个现有的独立服务器变成一个只有一个节点的复制集，那么必须重启该服务器进程并初始化。所以，哪怕你只有一个服务器，也最好将其配置成单节点复制集，这样当你以后需要添加节点时，就不需要重启了。

注3：rs是一个含有复制集工具方法的全局变量，可以用rs.help()来查看它能提供的帮助。

假设我们现在了有了第四个节点，怎么把它加入到这个复制集呢，运行如下命令：

    > rs.add("192.168.1.34:27017")

相应的，要删除一个节点可以这么做：


    > rs.remove("server-4:27017")

需要注意的是，当你删除一个节点，你会得到一堆错误输出。这不是你的操作失败了，相反，是重配置成功了。当你重新配置一个复制集时，主服务器关闭所有连接，所以当前的shell就关闭了，不过它会重新连接的。另外，在重新配置期间，该复制集没有主节点。

如果想查看当前复制集的数据，执行如下命令：

    > rs.config()
    {
        "_id" : "spock",
        "version" : 3,
        "members" : [
            {
                "_id" : 0,
                "host" : "192.168.1.31:27017"
            },
            {
                "_id" : 1,
                "host" : "192.168.1.32:27017"
            },
            {
                "_id" : 2,
                "host" : "192.168.1.33:27017"
            }
        ]
    }
注意version属性，复制集初始version是1，每配置一次就加一。
    
除了用add和remove来修改复制集，还可以直接修改：
    
    > var config = rs.config()
    > config.members[0].host = "bluven-centos:27017"
    > rs.reconfig(config)
