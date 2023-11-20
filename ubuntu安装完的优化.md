#### 1.2.1 允许root登录

默认情况下ssh服务禁止root用户登录，需要修改`/etc/ssh/sshd_config`将`PermitRootLogin`注释打开并配置为`yes`，并重启sshd服务

```bash
systemctl restart sshd.service
```

修改root密码

```bash
sudo passwd root
```

#### 1.2.2 设置时区

将时区配置为`Asia/Shanghai`

```bash
timedatectl set-timezone Asia/Shanghai
```

#### 1.2.3 禁用ufw防火墙服务

```bash
systemctl disable ufw.service --now
```

#### 1.2.4 文件打开数限制

```bash
cat << EOF >> /etc/security/limits.conf
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

#### 1.2.5 内核参数优化

编辑`/etc/sysctl.conf`文件，添加如下内容

```bash
# 开启IP转发.
net.ipv4.ip_forward = 1
 
# /proc/sys/fs/may_detach_mounts 默认设置为0
# 当系统有容器运行的时候，需要将该值设置为1
fs.may_detach_mounts = 1

# Default: 30
# 0 - 任何情况下都不使用swap。
# 1 - 除非内存不足（OOM），否则不使用swap。
vm.swappiness = 0

# 内存分配策略
# 0 - 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
# 1 - 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
# 2 - 表示内核允许分配超过所有物理内存和交换空间总和的内存
vm.overcommit_memory=1

# OOM时处理
# 1关闭，等于0时，表示当内存耗尽时，内核会触发OOM killer杀掉最耗内存的进程。
vm.panic_on_oom=0


# 最大文件句柄数
fs.file-max=52706963

# 最大文件打开数
fs.nr_open=52706963

# 设置 conntrack 的上限
net.netfilter.nf_conntrack_max=2310720

# TCP连接keepalive的持续时间，默认7200
net.ipv4.tcp_keepalive_time = 600

# TCP keepalive探测包重试次数
net.ipv4.tcp_keepalive_probes = 3

# TCP keepalive探测包发送间隔
net.ipv4.tcp_keepalive_intvl =15

# 内核中管理 TIME_WAIT 状态的数量
net.ipv4.tcp_max_tw_buckets = 36000

# 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭
net.ipv4.tcp_tw_reuse = 1

# 不属于任何进程的tcp socket最大数量. 超过这个数量的socket会被reset, 并告警
net.ipv4.tcp_max_orphans = 327680

# TCP FIN报文重试次数
net.ipv4.tcp_orphan_retries = 3

# 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭
net.ipv4.tcp_syncookies = 1

# 表示SYN队列的长度，默认为1024
net.ipv4.tcp_max_syn_backlog = 16384

# iptables防火墙使用了ip_conntrack内核模块实现连接跟踪功能
# 调整conntrack表最大数量
net.ipv4.ip_conntrack_max = 65536

# 调整系统中处于 SYN_RECV 状态的 TCP 连接数量
net.ipv4.tcp_max_syn_backlog = 16384

# 禁用tcp时间戳
# 默认为1,设为0关闭
net.ipv4.tcp_timestamps = 0

# 定义了系统中每一个端口最大的监听队列的长度,这是个全局的参数,默认值为128.限制了每个端口接收新tcp连接侦听队列的大小
net.core.somaxconn = 16384
```

使配置生效

```bash
sysctl --system
```

#### 1.2.6 网卡名降级

使网卡名变为ethx格式，编辑`/etc/default/grub`，将`GRUB_CMDLINE_LINUX`的值添加`net.ifnames=0 biosdevname=0`

```bash
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'

GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
```

更新grub配置文件

```bash
update-grub
```

修改网卡配置文件，编辑`/etc/netplan/00-installer-config.yaml`，将网卡名称修改为eth0

重新引导操作系统，即重启服务器

```bash
reboot
```

#### 1.2.7 修改apt国内镜像

修改文件

```bash
sed -i 's@//.*archive.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list
```

```bash
sed -i 's/security.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
```

更新apt缓存

```bash
apt update
```

#### 1.2.8  配置ntp自动对时

安装ntp服务

```bash
apt install ntp -y
```

禁用`systemd-timesyncd.service`服务

```bash
systemctl disable systemd-timesyncd.service --now
```

修改ntp配置文件`/etc/ntp.conf`，如果内网中有ntp服务器，应该修改为内网ntp服务器ip地址，如果服务器可以连接公网，则配置为阿里云公共ntp服务

配置ntp服务开机自启动

```bash
systemctl enable ntp.service --now
```