---
category: 数据库
---

## 缘由

昨天需求需要在专题新增一个字段， 可在后台进行修改。修改完毕后部署是下午开始的时候，仅做了对应的功能测试，事实上这是万恶之源。之后运营便开始在系统上设置该字段，到了第二天早上产品说出问题了，所有修改过该字段的对象，关联的图片都只剩6张。

分析了一下，大概就知道问题所在。方便运营修改，开关设置在列表上操作，修改的时候前端提交的数据是所有的内容，包括需要关联的图片。然饿，列表返回的数据并不是全的，关联的图片只返回前6张，最终导致悲剧。

## 想办法

从数据上看，修改过字段的专题跟图片都在，只是它们之间的关联被修改了，也就是多对多关系的中间表。这个表并没有做软删除，所以修改操作都会先删除之前的关联，再新增新的关联。

只能从以前的备份恢复了。还好，现在每天做三次数据库备份，昨天12点有备份一次，并且在运营操作之前。由于只是中间的关联表需要修改，其他的都是正常。 postgresql的pg_restore有只恢复指定表的选项,  `-t tablename` 。

然后12点之后关联表的数据要正常恢复，寻找一番，发现postgresql提供了一个copy表的方法，可以将一个表的数据copy到一个文件，也可以从一个文件copy回来。

```sql
-- 把12点后新关联的数据提取到一个临时表：
create table tmp_gif_topics as select * from gif_topics where gif_topics.topic_id > 3432;
-- 因为id 3432是12点前创建的专题，故12点后进行的关联数据就是需要的。

-- copy到文件中:
copy tmp_gif_topics to '/tmp/tmp_gif_topics.txt' with delimiter as '|';

-- 删掉临时表：
drop table tmp_gif_topics;

-- 执行数据恢复, 只恢复一个表:
`gunzip -c ${yesterday} | pg_restore -d database_name -t gif_topics -h 127.0.0.1 --clean -p 5432 -U username -O`

-- 进入数据库，把之前copy的内容恢复回来：
copy gif_topics from '/tmp/tmp_gif_topics.txt' with delimiter as '|';
```


## 问题

恢复完毕后，数据基本上正常了。然饿运营马上报了bug，专题没法发布了。 查看sentry的错误栈，在写入关联表时postgresql报错。
```
PG::NotNullViolation: ERROR: null value in column "id" violates not-null constraint DETAIL: Failing row contains (null, 3463, 34630, 104, 2017-07-20 07:55:13.696013, 2017-07-20 07:55:13.696013, null, 0)
INSERT INTO "gif_topics" ("topic_id", "gif_id", "user_id", "created_at", "updated_at") VALUES ($1, $2, $3, $4, $5)
```

id列明明是自动增长的，不需要显示指定值，没错。bug是一直存在的，后来仔细检查数据库表，发现在postgresql中还有一种类型是`sequence`，一般用作id增长的方式。

也就是`gif_topics`表的id列默认值没有了，其他表的id列都有DEFAULT 。应该是恢复表的时候这些限制被删掉了。

```sql
-- 先创建一个`SEQUENCE`：
CREATE SEQUENCE gif_topics_id_seq START 1 RESTART 144444;

-- 修改id列默认值:
ALTER TABLE gif_topics ALTER id SET DEFAULT nextval('gif_topics_id_seq'::regclass);
```

此问题解决，该表数据就可以插入了。

过了一会，运营又报另外的问题，无法将专题的图片排序（排序需要写入gif_topics表中的weight字段）。查看程序的异常栈，在gif_topics的对象save!方法中报'nil class no to_sym method'，哪里来的这东西。

在测试环境还没法模拟出来，只能上正式机器启动rails c命令行测试。按照程序的方式通过`topic_id`和`gif_id`拿到一个gif_topic对象，修改一个`weight`值，保存，bug重现了。在一堆的rails程序栈中，根本没法确定原因。

在一次打印gif_topic对象中发现没有id值，直接打印id值是可以的，但没法使用reload方法。究其原因，应该是无法通过主键定位对象。再次查看一下`gif_topics`表的信息，的确没有主键了。而且与以前相比许多index和foreign key都没有了。

```sql
-- 先把主键设置：
ALTER TABLE ONLY gif_topics ADD CONSTRAINT "gif_topics_pkey" PRIMARY KEY(id);
```

陆续把其他index和外健限制都加上，最终问题解决了。

## 总结

此次恢复数据的办法是可行的，但是测试不够全面，恢复后还是出了不少问题。并且查找问题的过程在程序上消耗过长，没有找到问题点（数据库对比），如果早点对比恢复前后的数据库表的情况，应该是很快能找到问题。

另外一个是使用pg_restore恢复的时候，虽然指定恢复一个表，但是恢复后之前表的信息均不见了（使用了选项--clean 删除之前的表）。所以恢复单表的时候还应该恢复表的信息。可以使用`-I index`选项恢复index。

使用`-t tablename` 指定多个表，包括sequence。
