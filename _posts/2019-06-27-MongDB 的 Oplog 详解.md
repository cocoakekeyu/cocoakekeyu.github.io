Oplog 是 MongoDB 实现复制集的关键数据结构，在复制集中 Primary 对数据库操作之后就会产生一个 Oplog 文档保存在 local.oplog.rs 集合中，Secondary 成员会拉取 Primary 的 Oplog 并重放相同的操作，从而达到 Secondary 成员与 Primary 有一致的数据。实际上复制集中每一个成员都会保存 Oplog，其他成员会根据连接延迟等因数选择最近的成员拉取 Oplog 数据。

Oplog 存在集合 local.oplog.rs，这是系统内置集合，一个 capped collection，即是这个 collection 有固定大小，一旦写满数据会从头开始写入，就像一个圆形的队列结构。这个 collection 大小在初始化集群时设置，默认的大小是 5% 的空闲磁盘空间，也可以在配置文件设置 oplogSizeMB 选项，或者在启动 MongoDB 后使用 replSetResizeOplog 命令动态设置 collection 大小。

Oplog 与 MongoDB 的其他的文档没有什么不同，它固定有一些属性：

- `ts`: MongoDB 的内置的特殊时间戳数据结构，如 Timestamp(1503110518, 1), 由秒级的 Unix 时间戳和一个顺序增长的整数 increment 表示。长度为 64 位，其中 Unix 时间戳占 32 位，后 32 位可以保存同一秒内的第几次操作。
- `h`: hash 值代表每个 Oplog 的唯一标识。
- `v`: Oplog 版本
- `ns`: namespace 命名空间，数据库+集合，用 database.collection 表示。但如果是表操作命令等，变成 database.$cmd。
- `op`：operation type，操作类型，包含以下几种：
  - `i`: insert, 插入文档
  - `u`: update, 更新文档
  - `d`: delete, 删除文档
  - `c`: command, 操作命令，如 createIndex 等
  - `n`: 空操作，用于空闲时主从同步 Oplog 时间信息

- `o`: operation， Oplog 操作的具体内容，例如 `i` operation type，`o` 即是插入的文档。对于 `u` operation type, 只更新部分内容， `o` 键的内容为 `{$set: {...}}`
- `o2`: 用于 update 操作，包含 `_id ` 属性值。

Oplog 的重放是幂等(idempotent)的，即是说同一个 Oplog 重放多次最终结果还是一致的。这是 MongoDB 将许多命令操作进行了转化，保持生成的 Oplog 是可以幂等的，如执行以下 $inc 操作：

```
db.test.update({_id: ObjectId("533022d70d7e2c31d4490d22")}, {$inc: {count: 1}})
```

产生的 Oplog 为:

```
{
  "ts" : Timestamp(1503110518, 1),
  "t" : NumberLong(8),
  "h" : NumberLong(-3967772133090765679),
  "v" : NumberInt(2),
  "op" : "u",
  "ns" : "mongo.test",
  "o2" : {
    "_id" : ObjectId("533022d70d7e2c31d4490d22")
  },
  "o" : {
    "$set" : {
      "count" : 2.0
    }
  }
}

```

以上 MongoDB 可以保证 Oplog 的数据操作(DML 语句)是幂等的，但数据表操作(DDL 语句)命令无法保证，例如重复执行相同的 createIndex 命令。

## Oplog 的查询

Capped collection 内文档是以插入顺序排序的，没有其他索引，但是 local.oplog.rs 是一个特殊的 capped collection，在 Wiredtiger 引擎的话，Oplog 的时间戳会作为一个特殊的元信息存储，使得 Oplog 可以以 ts 字段排序，查询 Oplog 时可以利用 ts 字段筛选。

一般来说 Secondary 同步需要经过 initial sync 和 incremental sync，initial sync 同步完成后，需拉取从同步时间点开始之后的 Oplog 进行持续重放。所以查询 Oplog 的操作一般是：

```
db.oplog.rs.find({$gte:{'ts': Timestamp(1503110518, 1)}})
```

Secondary 需要不断获取 Primary 产生的 Oplog, 复制集会使用 tailable cursor 持续获取 Oplog 数据，非常类似 Unix 系统的 tail -f。这会提高效率，因为一般的 cursor 使用完毕后就会关闭，而 tailable cursor 会保存上次的 id, 并持续获取数据。

如果使用 pymongo 驱动器，则定位从某个时间点之后的 Oplog 可以这麽写：

```Python

coll = db['local'].get_collection(
  'oplog.rs',
  codec_options=bson.codec_options.CodecOptions(document_class=bson.son.SON))

cursor = coll.find({'ts': {'$gte': start_optime}},
  cursor_type=pymongo.cursor.CursorType.TAILABLE,
  oplog_replay=True,
  no_cursor_timeout=True)

while True:
  try:
    oplog = cursor.next()
    process(oplog)
  except StopException:
    # 没有更多的 Oplog 数据
    time.sleep(1)

```

cursor_type 使用 TAILABLE 或者 TAILABLE_AWAIT，使用后一种类型时，如果没有更多的 Oplog 数据，则这次请求会阻塞等待有 Oplog 数据或者到达等待的时间超时返回。

设置 oplog_replay 标记可以表示此次请求的类型是保存 Oplog 的 capped collection, 提供 ts 筛选参数, 进行查询优化。

获取到 Oplog 之后，就可以做数据同步或者分发到感兴趣的消费者作特殊分析，如 MongoShake 工具。


参考了文档：

- Replica Set Oplog: https://docs.mongodb.com/manual/core/replica-set-oplog/
- MongoDB oplog 漫谈: http://caosiyang.github.io/2016/12/24/mongodb-oplog/
- MongoDB复制集原理: https://yq.aliyun.com/articles/64?spm=a2c4e.11155435.0.0.7c9a4547xnNydv
