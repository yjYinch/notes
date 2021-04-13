# Redis

## 一、安装

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

