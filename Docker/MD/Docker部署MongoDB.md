# Docker部署MongoDB

官方镜像地址：[mongo](https://hub.docker.com/_/mongo)

## 拉取镜像

```bash
docker pull mongo:4.2.8
```

## 创建目录

```bash
cd /opt
mkdir -p mongo/mongo_configdb
mkdir -p mongo/mongo_db
```

## 启动运行

```bash
docker run -d -it \
 --name mongo_4.2.8 \
 -p 27017:27017 \
 -v /opt/mongo/mongo_configdb:/data/configdb -v /opt/mongo/mongo_db:/data/db \
 mongo:4.2.8 \
 --auth
```

命令说明：

- --name 自定义别名
- -p 27017:27017 将容器的27017端口映射到主机的27017端口
- --auth：开启密码授权访问

## 为MongoDB添加管理员用户

进入MongoDB，为MongoDB添加管理员用户

```bash
docker exec -it mongo_4.2.8 mongo admin
```

在mongo命令行输入命令创建管理员账户

```bash
db.createUser({ user: 'root', pwd: 'root_password', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });
```

验证

```bash
# 测试下是否正确
db.auth("root", "root_password");
# 返回1表示正确
1
# 退出
exit;
```

## 创建访问指定数据库的用户

假设我们为 test 库创建一个用户，用户名为 test，密码为 test_password

切换到test库（如不存在会自动创建）

```bash
use test
```

创建test库下的用户

```bash
db.createUser({ user: 'test', pwd: 'test_password', roles: [{ role: "readWrite", db: "test" }] });
```

对 test 进行身份认证：

```bash
db.auth("test","test_password");
```

切换数据库

```bash
use test
```

添加数据

```bash
db.test.save({name:"zhangsan"});
```

## 其他

role角色参数参考：

- Read：允许用户读取指定数据库
- readWrite：允许用户读写指定数据库
- dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
- userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
- clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限
- readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
- readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
- userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限
- dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限
- root：只在admin数据库中可用。超级账号，超级权限

mongodb 有哪些权限:

1. 数据库用户角色：read、readWrite;
2. 数据库管理角色：dbAdmin、dbOwner、userAdmin；
3. 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
4. 备份恢复角色：backup、restore；
5. 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
6. 超级用户角色：root (这里还有几个角色间接或直接提供了系统超级用户的访问(dbOwner 、userAdmin、userAdminAnyDatabase))
7. 内部角色：__system

## 参考

- [Docker版MongoDB的安装](https://www.jianshu.com/p/2181b2e27021)
