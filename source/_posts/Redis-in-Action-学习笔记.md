title: Redis in Action 学习笔记
date: 2016-07-09 22:11:39
tags:
- redis
---
##### 几种常见的数据库的比较
|Name|Type|Data storage options|Query types|Additional features|
|---|---|---|---|---|
|Redis|In-memory non-relational database|Strings, lists, sets, hashes, sorted sets|Commands for each data type for common access patterns, with bulk oper- ations, and partial trans- action support|Publish/Subscribe, master/slave replica- tion, disk persistence, scripting (stored proce- dures）|
|memcached|In-memory key-value cache|Mapping of keys to values|Commands for create, read, update, delete, and a few others|Multithreaded server for additional perfor- mance|
|MySQL|Relational database|Databases of tables of rows, views over tables, spatial and third-party extensions|SELECT, INSERT, UPDATE, DELETE, functions, stored procedures|ACID compliant (with InnoDB), master/slave and master/master replication|
|MongoDB|On-disk non-relational document store|Databases of tables of schema-less BSON documents|Commands for create, read, update, delete, conditional queries, and more|Supports map-reduce operations, master/ slave replication, shard- ing, spatial indexes|

##### Redis支持的几种数据类型及对应的操作
* **Strings** SET/GET/DEL 这里的String可以用来存储
	* Byte string values
	* Integer vlues
	* Float values
* **Lists**	RPUSH/LRANGE/LINDEX/LPOP
* **Sets** SADD/SMEMBER/SISMEMBER/SREM
* ***Hashes* HSET/HGET/HGETALL/HDEL
* **Sorted sets** ZADD/ZRANGE/ZRANGEBYSCORE/ZREM

##### 两种持久化方式
* Snapshot
* Append-only file
* 
