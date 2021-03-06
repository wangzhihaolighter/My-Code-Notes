# Docker部署Nexus3（Maven私服）

Nexus3 Docker官网镜像地址：[sonatype/nexus3](https://hub.docker.com/r/sonatype/nexus3)

## 下载镜像

```bash
docker pull sonatype/nexus3:3.20.1
```

## 创建目录

```bash
mkdir -p ~/nexus/nexus-data  && chown -R 200 ~/nexus/nexus-data
```

## 启动容器

```bash
docker run -d -it -p 8081:8081 --name nexus3 -v ~/nexus/nexus-data:/nexus-data sonatype/nexus3:3.20.1
```

## 使用nexus3

### 登录

通过访问<http://IP:8081>进入管理平台。

新版本的初始登录密码已经不是admin123了，而是在容器中的/nexus-data/admin.password中

```bash
cat ~/nexus/nexus-data/admin.password
```

点击login，输入账号`admin`，密码则是刚才文件中的密码即可登录管理平台。
