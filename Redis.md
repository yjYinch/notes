# Redis

<font color=red size=4>***安装***</font>

1 官网下载安装包 [redis](https://redis.io/download) 或者获取redis的下载连接

2 将安装包拷贝到服务器或者使用命令下载

```bash
wget https://download.redis.io/releases/redis-6.2.1.tar.gz

tar -zxvf redis-6.2.1.tar.gz

#移动且重命名为redis
mv redis-6.2.1 /usr/local/redis

make
```

3 启动redis

```bash
src/redis-server
```



<font color=red size=4>***常用命令***</font>

`client list`: 表示列举连接当前redis服务的所有客户端信息。



---

<font color=red size=4>***缓存击穿、雪崩、穿透***</font>

**什么是缓存击穿？**

大量的请求访问某个key，而与此同时该key突然失效，导致请求全都转移到数据库，会导致数据库挂了。

**解决方案：**

操作数据库的时候，加锁。

**什么是缓存雪崩？**

多个key同时失效，大量的请求访问多个key。

**解决方案：**

让多个key不同时失效，给热点key设置不同的失效时间

**什么是缓存穿透？**

有大量的请求访问时，只有少部分的key在缓存中存在，而有大量的key不存在，这样请求也会直接访问到数据库，也会导致数据库扛不住压力而挂掉。这种情况往往是黑客伪造请求，发起的恶意攻击。

**解决方案：**

根据业务规则过滤key。



---

<font color=red size=4>***Redis交互操作***</font>

与redis交互对象有四种

* **客户端**
* **磁盘**
* **主从节点**
* **切片集群实例**

**客户端：**

（1）网络IO交互

（2）键值对增删改查

（3）数据库操作

**磁盘：**

（1）生成RDB快照

（2）记录AOF日志

（3）AOF日志重写

**主从节点**：

（1）主库生成、传输RDB文件

（2）从库接收、加载RDB文件

（3）从库清空数据库

**切片集群实例：**

（1）传输哈希槽信息

（2）数据迁移操作



----

