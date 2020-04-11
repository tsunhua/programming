---
title: MongoDB
date: '2018-11-09T11:45:03.000Z'
tags:
  - Java
  - MongoDB
comments: true
---

# MongoDB

MongoDB 是一款开源的面向文档的数据库（document database）， [NoSQL](https://zh.wikipedia.org/zh-cn/NoSQL) 中一种，同样使用文档存储实现 NoSQL 的 DB 还有 MarkLogic、OrientDB、CouchDB 等等。

## 安装

### 安装在 Mac 中

Mac 用户可以直接使用 Homebrew 安装，命令如下：

```text
sudo brew install mongodb
```

也可以自己到 [MongoDB 的下载中心](https://www.mongodb.com/download-center/community) 下载对应的系统和版本，如果是 Linux 的话可以使用 `wget` 下载：

```text
wget "https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-4.0.4.tgz"
```

并配置环境变量，如下：

```text
export PATH={MONGODB_DIR}/bin:$PATH
```

### 安装在 CentOS

```bash
cat > /etc/yum.repos.d/mongodb-org.repo <<EOF
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc
EOF

yum repolist

sudo yum install mongodb-org

sudo systemctl start mongod

sudo systemctl enable mongod

# 修改配置文件/etc/mongod.conf 后需要执行以下语句重载
sudo systemctl reload mongod

# 查看日志
sudo tail /var/log/mongodb/mongod.log
```

## 启动

```text
# {mongo_db_file_path} 为指定的数据库文件存放位置，不支持~符号。如果使用默认位置 /data/db ，也需要先手动创建。
$ mongod --dbpath={mongo_db_file_path} --bind_ip_all --fork --logpath ./mongo.log
```

## 终端连接

（1）本地连接

```text
$ mongo
```

（2）远程连接

```text
$  mongo 172.2.0.3:27017
```

## 基本概念

### BSON

MongoDB 的文件存储格式为 BSON，所谓 BSON，即是 Binary JSON，为 JSON 文档对象的二进制编码格式，其扩展了 JSON 的数据类型，支持浮点数和整数的区分，支持日期类型，支持直接存储 JS 的正则表达式，支持 32 位和 64 位数字的区分，支持直接存储 JS 函数等等。看起来 BSON 对于 JS 还是挺友好的呵。

> 注意：从 Shell 终端中输入的数值都会被存储为 64 位浮点类型。

### 文档（Document）

一个文档就相当于关系型数据库中的**行**的概念，由多个键值有序组成，格式为 BSON。示例如下：

```javascript
{ "_id" : ObjectId("5c05e74a65a27abc9a619f8a"), "a_key" : "a_value" }
```

其中 `_id` 是系统自动生成的键，当然也可以在创建时自定义值。

### 集合（Collection）

一个集合就相当于关系型数据库中的**表**的概念，由多个文档组成。集合中的默认索引为 `_id` ，可以新建其他键的索引来优化查询，MongoDB 支持单字段索引（Single Field Index）、复合索引（Compound Index）以及多键索引（Multikey Index）等等，可以根据需求进行选用。

### 数据库（Database）

多个集合组成一个数据库，不同的数据库之间文件是隔离的。单个 MongoDB 实例可以容纳多个独立数据库。默认系统存在以下的保留数据库：

1. admin：用户权限相关
2. local：存储限于本地的集合
3. config：分片配置相关

## Shell 操作

### database 级别

```text
# 列出所有的数据库
> show dbs

# 查看当前使用的数据库
> db

# 切换当前使用的数据库
> use a_db

# 创建数据库
> use new_db

# 删除数据库
> db.dropDatabase()
```

### collection 级别

```text
# 显示数据库中的所有 collection
> show collections

# 列出 collection 中的所有列
> db.a_collection.find()

# 删除 collection
> db.a_collection.drop()

# 新建 collection
> db.createCollection("new_collection")

# 重命名 collection
> db.a_collection.renameCollection("new_name")

# 清空 collection 中数据
> db.a_collection.drop({})
```

### docuement 级别

```text
# 插入文档
> db.a_collection.insert({
    "a_key": "a_value",
    "b_key": 100,
    "c_key": true
})

# 列出所有文档，并美化
> db.a_collection.find().pretty()

# 查询记录条数
> db.a_collection.find().count()

# 略过前100条
> db.a_collection.find().skip(100)

# MongoDB AND 且过滤器
> db.a_collection.find({
    "a_key": "a_value",
    "c_key": true
})
# MongoDB OR 或过滤器
> db.a_collection.find({
   $or:[
       { "a_key": "a_value"},
       { "a_key": "another_value" }
   ]
})
# MongoDB 投影，只返回指定的字段
> db.a_collection.find({},{"a_key", "c_key"})
# 查询存在某字段的文档
> db.a_collection.find({"a_key",{"$exists":true}})


# 更新单个文档
db.a_collection.update({"a_key": "a_value"},{$set:{"a_key": "another_value"}})
# 更新多个文档
db.a_collection.update({"a_key": "a_value"},{$set:{"a_key": "another_value"}},,{multi: true})

# 删除文档
db.a_collection.remove({"a_key": "a_value"})

# 后台执行创建单一复合索引操作
db.a_collection.createIndex({"a_key": 1,"c_key": -1},{unique: true,background: true})
# 查询所有索引
db.a_collection.getIndexes()
# 删除索引
db.a_collection.dropIndex({"a_key":1})
```

## Java 操作

### 模块划分

1. bson：高性能的编码解码。
2. mongodb-driver-core：核心库，抽取出来主要是用于自定义 API。
3. mongodb-driver-legacy：兼容旧的 API 的同步 Java Driver。
4. mongodb-driver-sync：只包含 MongoCollection 泛型接口，服从一套新的跨 Driver 的 CRUD 规范。
5. mongodb-driver：mongodb-driver-legacy + mongodb-driver-sync，新项目推荐使用它！
6. mongodb-driver-async：新的异步 API，充分利用 Netty 或者 Java7 的 AsynchronousSocketChannel 已达到快而非阻塞的 IO。
7. mongo-java-driver（uber-jar）：包含 bson,  mongodb-driver-core 和 mongodb-driver。

### 引入依赖

```groovy
dependencies {
    compile 'org.mongodb:mongodb-driver-sync:3.9.1'
}
```

### 在 v3.6.4 使用 MongoURI

```java
String mongoUri = ConfigManager.getInstance().getString(DistributedConfig.MONGODB_URI);
ConnectionString connectionString = new ConnectionString(mongoUri);
CodecRegistry pojoCodecRegistry = fromRegistries(MongoClient.getDefaultCodecRegistry(),
                                                 fromProviders(PojoCodecProvider.builder()
                                                               .automatic(true)
                                                               .build()));
MongoClient mongoClient = new MongoClient(new MongoClientURI(mongoUri,
                                                             MongoClientOptions.builder()
                                                             .codecRegistry(
                                                                 pojoCodecRegistry)));
String database = connectionString.getDatabase();
if (Strings.isNullOrEmpty(database)) {
    database = "my_db";
}
MongoDatabase mongoDatabase = mongoClient.getDatabase(database);
```

### 在  v3.9.1 使用 MongoURI

```java
public class Mongo {

  private MongoDatabase mongoDatabase;

  private Mongo() {
    String mongoUri = ConfigManager.getInstance().getString(DistributedConfig.MONGODB_URI);
    ConnectionString connectionString = new ConnectionString(mongoUri);
    CodecRegistry pojoCodecRegistry = fromRegistries(MongoClientSettings.getDefaultCodecRegistry(),
                                                     fromProviders(PojoCodecProvider.builder()
                                                                                    .automatic(true)
                                                                                    .build()));
    MongoClientSettings settings = MongoClientSettings.builder()
                                                      .applyConnectionString(connectionString)
                                                      .codecRegistry(pojoCodecRegistry)
                                                      .build();
    MongoClient mongoClient = MongoClients.create(settings);
    String database = connectionString.getDatabase();
    if (Strings.isNullOrEmpty(database)) {
      database = "my_db";
    }
   MongoDatabase mongoDatabase = mongoClient.getDatabase(database);
  }

  public <T> MongoCollection<T> getCollection(Class<T> documentClass) {
    return mongoDatabase.getCollection(CaseFormat.LOWER_CAMEL.to(CaseFormat.LOWER_UNDERSCORE,
                                                                 documentClass.getSimpleName()),
                                       documentClass);
  }

}
```

### 事务支持

MongoDB 的事务支持始于 MongoDB 4.0，对应 Java Driver 版本为 3.8.0，对应 Python 版本为 3.7.0，详情阅读 [Transactions and MongoDB Drivers - mongodb.com](https://docs.mongodb.com/manual/core/transactions/#transactions-and-mongodb-drivers).

```java
void runTransactionWithRetry(Runnable transactional) {
    while (true) {
        try {
            transactional.run();
            break;
        } catch (MongoException e) {
            System.out.println("Transaction aborted. Caught exception during transaction.");

            if (e.hasErrorLabel(MongoException.TRANSIENT_TRANSACTION_ERROR_LABEL)) {
                System.out.println("TransientTransactionError, aborting transaction and retrying ...");
                continue;
            } else {
                throw e;
            }
        }
    }
}

void commitWithRetry(ClientSession clientSession) {
    while (true) {
        try {
            clientSession.commitTransaction();
            System.out.println("Transaction committed");
            break;
        } catch (MongoException e) {
            // can retry commit
            if (e.hasErrorLabel(MongoException.UNKNOWN_TRANSACTION_COMMIT_RESULT_LABEL)) {
                System.out.println("UnknownTransactionCommitResult, retrying commit operation ...");
                continue;
            } else {
                System.out.println("Exception during commit ...");
                throw e;
            }
        }
    }
}

void updateEmployeeInfo() {
    MongoCollection<Document> employeesCollection = client.getDatabase("hr").getCollection("employees");
    MongoCollection<Document> eventsCollection = client.getDatabase("hr").getCollection("events");

    try (ClientSession clientSession = client.startSession()) {
        clientSession.startTransaction();

        employeesCollection.updateOne(clientSession,
                Filters.eq("employee", 3),
                Updates.set("status", "Inactive"));
        eventsCollection.insertOne(clientSession,
                new Document("employee", 3).append("status", new Document("new", "Inactive").append("old", "Active")));

        commitWithRetry(clientSession);
    }
}


void updateEmployeeInfoWithRetry() {
    runTransactionWithRetry(this::updateEmployeeInfo);
}
```

## 备份与还原

使用 Mongo 安装包 bin 目录下的 mongodump 进行备份，mongorestore 进行还原。

### 备份

```text
mongodump -h IP --port 端口 -u 用户名 -p 密码 -d 数据库 -o 文件存在路径
```

### 还原

```text
mongorestore -h IP --port 端口 -u 用户名 -p 密码 -d 数据库 --drop 文件存在路径
```

当为还原 bson 文件为

```text
mongorestore -h IP --port 端口 -u 用户名 -p 密码 -d 数据库 --drop bson文件路径 -d 表名
```

## 数据库迁移

```text
db.copyDatabase("db_to_rename","db_renamed","localhost")
```

## Q&A

### 一个服务中该使用一个还是多个 MongoClient？

通常一个服务应使用一个全局的 MongoClient，并且 MongoClient 中已经实现了一个连接池，最大值默认为 1000000 的连接限制，这相当于没有限制。

参考：[为什么 MongoDB 连接数被用满了？ - mongoing.com](http://www.mongoing.com/archives/3145)

### Invalid BSON field name id

更新文档时出现该错误，原因是使用了 `updateOne` 但是没有 `$set` 字段，改为使用 `replaceOne` 就不用这么麻烦了。

### readString can only be called when CurrentBSONType is STRING, not when CurrentBSONType is OBJECT\_ID

给名为 `id` 的字段添加注解 `@BsonProperty("id")` 即可。

### MongoWaitQueueFullException

错误日志：

```text
com.mongodb.MongoWaitQueueFullException: Too many threads are already waiting for a connection. Max number of threads (maxWaitQueueSize) o
f 500 has been exceeded.
    at com.mongodb.internal.connection.DefaultConnectionPool.createWaitQueueFullException(DefaultConnectionPool.java:280)
    at com.mongodb.internal.connection.DefaultConnectionPool.get(DefaultConnectionPool.java:99)
    at com.mongodb.internal.connection.DefaultConnectionPool.get(DefaultConnectionPool.java:92)
    at com.mongodb.internal.connection.DefaultServer.getConnection(DefaultServer.java:85)
    at com.mongodb.binding.ClusterBinding$ClusterBindingConnectionSource.getConnection(ClusterBinding.java:115)
    at com.mongodb.operation.OperationHelper.withReleasableConnection(OperationHelper.java:424)
    at com.mongodb.operation.MixedBulkWriteOperation.execute(MixedBulkWriteOperation.java:192)
    at com.mongodb.operation.MixedBulkWriteOperation.execute(MixedBulkWriteOperation.java:67)
    at com.mongodb.client.internal.MongoClientDelegate$DelegateOperationExecutor.execute(MongoClientDelegate.java:193)
    at com.mongodb.client.internal.MongoCollectionImpl.executeBulkWrite(MongoCollectionImpl.java:467)
    at com.mongodb.client.internal.MongoCollectionImpl.bulkWrite(MongoCollectionImpl.java:447)
    at com.mongodb.client.internal.MongoCollectionImpl.bulkWrite(MongoCollectionImpl.java:442)
```

## 参考

1. [MongoDB Driver Quick Start - mongoDB](https://mongodb.github.io/mongo-java-driver/3.9/driver/getting-started/quick-start/)
2. [MongoDB学习（二）：数据类型和基本概念 - Hejin.Wang](https://www.cnblogs.com/egger/archive/2013/04/27/3047191.html)
3. [MongoDB索引原理 - mongoing.com](http://www.mongoing.com/archives/2797)

