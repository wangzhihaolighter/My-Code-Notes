# Docker部署Gitlab

官方Docker hub中的Gitlab镜像地址：[gitlab-ce](https://hub.docker.com/r/gitlab/gitlab-ce)

## 下载镜像

```bash
# 不加 tag 则默认为最新版本 latest
docker pull gitlab/gitlab-ce
```

## 创建目录

通常会将 GitLab 的配置 (etc) 、 日志 (log) 、数据 (data) 放到容器之外， 便于日后升级， 因此请先准备这三个目录。

```bash
mkdir -p /opt/gitlab/config
mkdir -p /opt/gitlab/logs
mkdir -p /opt/gitlab/data
```

## 启动运行

```bash
docker run --detach \
  --hostname gitlab.example.com \
  --publish 8443:443 --publish 8880:80 --publish 8222:22 \
  --name gitlab \
  --restart always \
  --volume /opt/gitlab/config:/etc/gitlab \
  --volume /opt/gitlab/logs:/var/log/gitlab \
  --volume /opt/gitlab/data:/var/opt/gitlab \
  --privileged=true \
  gitlab/gitlab-ce:latest
```

说明:

- --hostname gitlab.example.com: 设置主机名或域名
- --publish 8443:443：将http：443映射到外部端口8443
- --publish 8880:80：将web：80映射到外部端口8880
- --publish 8222:22：将ssh：22映射到外部端口8222
- --name gitlab: 运行容器名
- --restart always: 自动重启
- --volume /opt/gitlab/config:/etc/gitlab: 挂载目录
- --volume /opt/gitlab/logs:/var/log/gitlab: 挂载目录
- --volume /opt/gitlab/data:/var/opt/gitlab: 挂载目录
- --privileged=true 使得容器内的root拥有真正的root权限。否则，container内的root只是外部的一个普通用户权限

执行命令后，可以使用下面的命令查看容器运行状态：

```bash
docker ps -a
```

看到容器的STATUS为Up时，部署成功。

## 访问

gitlab启动成功后，浏览器访问 <http://ip:8880> ，即可访问（这里ip为启动时设置的主机名或域名），配置号nginx，开放对应端口后即可访问。

首次访问需要为root用户设置密码，设置完成后需要登录，默认用户名为：root， 密码为刚刚设置的密码。

## 配置邮件服务器

想要让 GitLab 给你发送邮件，还要配置一下邮件服务器，这里以QQ邮箱的 IMAP/SMTP服务 来配置。

打开邮箱->设置->账户，然后开启 IMAP/SMTP服务，然后根据文档获取 授权码 ，这步比较重要。

然后跳转至挂载目录 /opt/gitlab/config/ 编辑gitlab.rb 文件，找到 Email Settings的注释位置，然后修改以下内容：

```conf
### Email Settings
gitlab_rails['smtp_enable'] = true # 开启 SMTP 功能
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 465 # 端口不可以选择587，测试过会发送邮件失败
gitlab_rails['smtp_user_name'] = "test@qq.com" # 你的邮箱账号
gitlab_rails['smtp_password'] = "" # 授权码，不是密码
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = 'test@qq.com' # 发件人信息，必须跟‘smtp_user_name’保持一致，否则报错
gitlab_rails['smtp_domain'] = "qq.com" # 修改并不影响 
```

配置完成后保存，然后输入下面的命令使配置生效。

```bash
sudo docker exec gitlab gitlab-ctl reconfigure
```

使配置生效之后我们可以使用 gitlab 自带的工具进行一下测试。依次执行下面的命令：

```bash
# 开启 gitlab 的 bash 工具
docker exec -it gitlab bash

# 开启 gitlab-rails 工具
gitlab-rails console production

# 发送邮件进行测试
Notify.test_email('test_001@123.com', 'Message Subject', 'Message Body').deliver_now
```

测试完成之后退出gitlab的bash工具，重启 gitlab 即可。

```bash
docker restart gitlab
```

## 配置 Git 仓库访问路径

在之前第一次运行 gitlab 容器的时候，有一个参数 hostname 为 gitlab.example.com , 如果配置了域名可以忽略这一步，如果你没有配置相应域名的话，你的仓库的地址将会变为下面这样：

```text
ssh : git@gitlab.example.com:test/test.git
http：gitlab.example.com/test/test.git
```

如果域名不存在的话，这个地址是无法进行 clone 的。

为了解决这个问题，我们可以设置成 IP 或 你配置了的域名来访问。

打开文件 /opt/gitlab/config/gitlab.rb 文件并找到：

```bash
# external_url 'GENERATED_EXTERNAL_URL'
```

这行，去掉注释，并按照下面的格式修改。

```bash
# ip 形式
external_url 'http://192.168.1.44'

# 域名形式
external_url 'http://JemGeek.com'

# 子域名
external_url 'http://gitlab.JemGeek.com'

# 其他形式
external_url 'http://JemGeek.com/gitlab'
```

以上形式都是可以的。修改完成后，输入命令：

```bash
docker exec gitlab gitlab-ctl reconfigure
```

使配置生效，然后重启 gitlab 即可。

## 配置ssh密钥后一直提示输入密码

GitLab 镜像启动后是占用容器的 22 端口，而我是使用宿主机的 8222 端口跟 GitLab 容器 22 端口进行的映射，主要是防止和我宿主机 22 端口冲突。想到问题的关键，解决就简单了，编辑 GitLab 配置文件，指定 SSH 端口为 8222 即可。

```bash
cd /opt/gitlab/config
vi gitlab.rb

gitlab_rails['gitlab_shell_ssh_port'] = 8222

#重启容器
docker restart gitlab
```

注意：服务器要开放8222端口。

## 设置gitlab push文件大小

1.修改 GitLab 配置文件，修改 nginx 的 client_max_body_size 配置。

```bash
cd /opt/gitlab/config
vi gitlab.rb

nginx['enable'] = true
nginx['client_max_body_size'] = '1024m'
nginx['redirect_http_to_https'] = false
nginx['redirect_http_to_https_port'] = 80

#重启容器
docker restart gitlab
```

2.在gitlab web管理页面：管理中心 -> 设置 -> 账户和限制，点击展开，修改 `最大推送大小 (MB)` 即可。

## 升级

参照官方的说明， 将原来的容器停止， 然后删除：

```bash
docker stop gitlab
docker rm gitlab
```

然后重新拉一个新版本的镜像下来，

```bash
docker pull gitlab/gitlab-ce
```

使用原来的运行命令运行：

```bash
docker run --detach \
  --hostname gitlab.example.com \
  --publish 8443:443 --publish 8880:80 --publish 8222:22 \
  --name gitlab \
  --restart always \
  --volume /opt/gitlab/config:/etc/gitlab \
  --volume /opt/gitlab/logs:/var/log/gitlab \
  --volume /opt/gitlab/data:/var/opt/gitlab \
  --privileged=true \
  gitlab/gitlab-ce:latest
```

如果，迁移了gitlab目录，注意复制文件夹的时候保留文件夹内文件的权限，如迁移到`/mnt`目录下：

```bash
mkdir /mnt/gitlab
cp -R -p /opt/gitlab /mnt/gitlab
```

GitLab 在初次运行的时候会自动升级， 为了预防万一， 还是建议先备份一下 /opt/gitlab/ 这个目录。

大版本升级（例如从 8.7.x 升级到 8.8.x）用上面的操作有可能会出现错误， 如果出现错误可以尝试登录到容器内部, 依次执行下面的命令：

```bash
gitlab-ctl reconfigure
gitlab-ctl restart
```

## 参考

- [docker安装gitlab](https://segmentfault.com/a/1190000019772866)
