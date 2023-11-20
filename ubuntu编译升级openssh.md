假设升级的目标主机不可以上网，演示离线情况下升级流程



## 1.下载所需依赖deb包

使用能够apt 正常安装的机器，将deb包下载到本地，然后将deb离线包传输到目标主机上

编译openssh所需依赖和gcc编译工具

```bash
apt install gcc make zlib1g-dev libpam0g-dev -d -y
```

将deb包传输到目标主机上并安装

```
dpkg -i *.deb
```

## 2.下载并编译openssl

下载指定版本openssl，并将其传输到目标机器上

```bash
https://www.openssl.org/source/old/3.1/openssl-3.1.0.tar.gz
```

 解压并编译

```bash
tar xf openssl-3.1.0.tar.gz
```

```bash
cd openssl-3.1.0
```

```bash
./config shared --prefix=/usr/local/openssl
```

```bash
make -j4 && make install  # 根据cpu核心数指定
```

配置动态链接库缓存

```bash
echo "/usr/local/openssl/lib64" > /etc/ld.so.conf.d/openssl.conf
```

更新动态链接库缓存

```bash
ldconfig -v
```

## 3.卸载旧版本openssh

在卸载旧版本之前需要先备份某些文件

```bash
cp /etc/passwd /root/
```

```bash
cp /etc/pam.d/sshd /root/
```

卸载旧版本openssh

```bash
apt purge openssh* -y
```

## 4.下载并编译openssh

下载指定版本openssh，并将其传输到目标机器上

```bash
https://mirrors.aliyun.com/pub/OpenBSD/OpenSSH/portable/openssh-9.3p1.tar.gz
```

解压并编译

```bash
tar xf openssh-9.3p1.tar.gz
```

```bash
cd openssh-9.3p1
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
install -m755 -d /usr/share/doc/openssh-9.3p1
install -m644 INSTALL LICENCE OVERVIEW README* /usr/share/doc/openssh-9.3p1
```

编辑 `/etc/ssh/sshd_config`，将注释掉的 `UsePAM no` 取消注释并改为 `yes`，并修改 `PermitRootLogin` 的值修改为 `yes`。

```conf
UsePAM yes
.....
PermitRootLogin yes
```

配置systemd启动文件

```bash
vim /lib/systemd/system/sshd.service
```

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

vim /lib/systemd/system/sshd-keygen.service

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
