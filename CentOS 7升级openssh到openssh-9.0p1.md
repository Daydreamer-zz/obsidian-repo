假设升级的目标主机不可以上网，演示离线情况下升级流程



## 1.下载所需依赖rpm包

使用能够yum正常安装的机器，将rpm包下载到本地，然后将rpm离线包传输到目标主机上

编译openssh所需依赖和gcc编译工具，以及telnet-server

```bash
yum install perl gcc-c++ make pam-devel zlib-devel telnet telnet-server xinetd --downloadonly --downloaddir=/root/upgrade_openssh
```



将rpm包传输到目标主机上并安装

```bash
rpm -ivh *.rpm --force
```

## 2.配置目标主机telnet-server

配置允许telnet root登录，在`/etc/securetty`下添加如下内容

```
pts/0
pts/1
```



启动服务

```bash
systemctl start xinetd.service telnet.socket
```

**防火墙应该放行tcp 23端口，然后通过局域网其他机器telnet连接到目标服务器进行后续操作**

## 3.下载并编译openssl

下载指定版本openssl，并将其传输到目标机器上

```bash
https://www.openssl.org/source/old/1.1.1/openssl-1.1.1s.tar.gz
```



 解压并编译

```bash
tar xf openssl-1.1.1s.tar.gz
```

```bash
cd openssl-1.1.1s
```

```bash
./config shared --prefix=/usr/local/openssl
```

```bash
make -j4 && make install  # 根据cpu核心数指定
```



配置动态链接库缓存

```bash
echo "/usr/local/openssl/lib" > /etc/ld.so.conf.d/openssl.conf
```



更新动态链接库缓存

```bash
ldconfig -v
```

## 4.卸载旧版本openssh

通过rpm卸载旧版本openssh

```bash
rpm -e openssh openssh-clients openssh-server --nodeps
```



备份原配置文件

```bash
mv /etc/ssh /etc/ssh_old
```

## 5.下载并编译openssh

下载指定版本openssh，并将其传输到目标机器上

```bash
https://mirrors.aliyun.com/pub/OpenBSD/OpenSSH/portable/openssh-9.0p1.tar.gz
```



解压并编译

```bash
tar xf openssh-9.0p1.tar.gz
```

```bash
cd openssh-9.0p1
```

```bash
./configure --prefix=/usr --sysconfdir=/etc/ssh --with-ssl-dir=/usr/local/openssl --with-pam --with-md5-passwords
```

```bash
make -j4 && make install  # 根据cpu核心数指定
```



在当前openssh源码包目录执行如下命令，安装额外文件

```bash
install -m755 contrib/ssh-copy-id /usr/bin
install -m644 contrib/ssh-copy-id.1 /usr/share/man/man1
install -m755 -d /usr/share/doc/openssh-9.0p1
install -m644 INSTALL LICENCE OVERVIEW README* /usr/share/doc/openssh-9.0p1
```



编辑 `/etc/ssh/sshd_config`，将注释掉的 `UsePAM no` 取消注释并改为 `yes`，并修改 `PermitRootLogin` 的值修改为 `yes`。



配置PAM，新建`/etc/pam.d/sshd`，写入如下内容

```
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```



配置systemd启动文件

/lib/systemd/system/sshd.service

```
[Unit]
Description=OpenSSH Daemon
Wants=sshdgenkeys.service
After=sshdgenkeys.service
After=network.target

[Service]
ExecStart=/usr/sbin/sshd -D
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=always

[Install]
WantedBy=multi-user.target
```

/lib/systemd/system/sshd-keygen.service

```
[Unit]
Description=SSH Key Generation
ConditionPathExists=|!/etc/ssh/ssh_host_dsa_key
ConditionPathExists=|!/etc/ssh/ssh_host_dsa_key.pub
ConditionPathExists=|!/etc/ssh/ssh_host_ecdsa_key
ConditionPathExists=|!/etc/ssh/ssh_host_ecdsa_key.pub
ConditionPathExists=|!/etc/ssh/ssh_host_ed25519_key
ConditionPathExists=|!/etc/ssh/ssh_host_ed25519_key.pub
ConditionPathExists=|!/etc/ssh/ssh_host_rsa_key
ConditionPathExists=|!/etc/ssh/ssh_host_rsa_key.pub

[Service]
ExecStart=/usr/bin/ssh-keygen -A
Type=oneshot
RemainAfterExit=yes
```



启动sshd

```bash
systemctl daemon-reload
```

```bash
systemctl enable sshd --now
```



## 6.关闭telnet-server并通过ssh连接测试

关闭telnet-server服务

```bash
systemctl stop xinetd.service telnet.socket
```



尝试通过ssh连接到目标主机上

