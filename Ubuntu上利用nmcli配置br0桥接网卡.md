## 1. 创建br0网桥

```bash
sudo nmcli connection add ifname br0 type bridge con-name br0
```

```bash
sudo nmcli connection add type bridge-slave ifname eth0 master br0 #可能不是eth0
```

## 2. 为网桥启用或者禁用STP(可选)

```bash
sudo nmcli connection modify br0 bridge.stp no
```

## 3.为网桥配置ipv4静态地址

```bash
sudo nmcli connection modify br0 ipv4.method manual
```

```bash
sudo nmcli connection modify br0 ipv6.method disabled
```

```bash
sudo nmcli connection modify br0 ipv4.addresses '192.168.10.183/24'
```

```bash
sudo nmcli connection modify br0 ipv4.gateway '192.168.10.1'
```

```bash
sudo nmcli connection modify br0 ipv4.dns '192.168.10.1'
```

## 4. 删除eth0

如果删除有问题，可以通过nmtui图形页面删除

```bash
sudo nmcli connection delete eth0 #可能不是eth0
```

## 5.启动网桥

```bash
sudo nmcli connection up br0
```

配置内核参数

```bash
cat >> /etc/sysctl.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
EOF
```