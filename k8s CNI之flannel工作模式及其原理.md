# k8s CNI之flannel工作模式及其原理



## 一.Flannel工作模式

Flannel支持多种数据转发方式

- UDP：最早支持的异种方式，由于性能最差，目前已经弃用
- VXLAN：Overlay Network方案，源数据包封装在另一种网络包里面进行路由转发和通信
- Howt-GW：Flannel通过在各个节点上的Agent进程，将容器网络的路由信息刷新到主机的路由表上，这样一来所有的主机都有整个容器网络的路由数据了
- Directrouting：兼顾vxlan和host-gw工作

## 二.VXLAN工作模式原理

本例k8s pod网段为`172.16.0.0/12`，node网段为`192.168.10.0/24`

cni0是一个Linux虚拟网桥，容器与cni0之间通过veth-pair连接。

flannel.1是一个Linux虚拟网络设备，类型为VXLAN，Flannel为了能够在二层网络上打通隧道，VXLAN模式会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备叫做VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。

```bash
[root@k8s-node01 ~]# ifconfig flannel.1
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 172.16.2.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::2056:a3ff:fe21:8b8d  prefixlen 64  scopeid 0x20<link>
        ether 22:56:a3:21:8b:8d  txqueuelen 0  (Ethernet)
        RX packets 63583531  bytes 83578209323 (77.8 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 62114167  bytes 16007611743 (14.9 GiB)
        TX errors 0  dropped 311 overruns 0  carrier 0  collisions 0
```

创建示例pod

```bash
|10:44:31|root@node01:[~]> kubectl delete deployments.apps nginx 
deployment.apps "nginx" deleted
|10:44:35|root@node01:[~]> kubectl create deployment nginx --image nginx:alpine --replicas 2
deployment.apps/nginx created
|10:45:39|root@node01:[~]> kubectl get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
nginx-d7b6f6c9c-49xl6   1/1     Running   0          3s    172.16.3.32   k8s-node02   <none>           <none>
nginx-d7b6f6c9c-szchl   1/1     Running   0          3s    172.16.2.30   k8s-node01   <none>           <none>
```

如果pod nginx-d7b6f6c9c-49xl6 访问pod nginx-d7b6f6c9c-szchl，源地址`172.16.3.32`，目的地址`172.16.2.30 `，数据包传输流程如下

### 1.容器路由

进入nginx-d7b6f6c9c-49xl6容器，查看容器路由

可以看到容器默认网关下一跳为`172.16.3.1`

```bash
|10:47:56|root@node01:[~]> kubectl exec -it nginx-d7b6f6c9c-49xl6 -- route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.3.1      0.0.0.0         UG    0      0        0 eth0
172.16.0.0      172.16.3.1      255.240.0.0     UG    0      0        0 eth0
172.16.3.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

### 2.主机路由

容器的默认网关为`172.16.3.1`，即为宿主机的cni0网卡

```bash
[root@k8s-node02 ~]# ifconfig cni0
cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 172.16.3.1  netmask 255.255.255.0  broadcast 172.16.3.255
        inet6 fe80::8c3d:3eff:fee8:6645  prefixlen 64  scopeid 0x20<link>
        ether 8e:3d:3e:e8:66:45  txqueuelen 1000  (Ethernet)
        RX packets 95555389  bytes 61810591718 (57.5 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 104031121  bytes 150822839103 (140.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

查看宿主机路由表，可以看到，目的地址为`172.16.2.30` ，数据包应该转发给`flannel.1`虚拟网卡，也就是来到了隧道的入口

```bash
[root@k8s-node02 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.10.254  0.0.0.0         UG    100    0        0 eth0
172.16.0.0      172.16.0.0      255.255.255.0   UG    0      0        0 flannel.1
172.16.1.0      172.16.1.0      255.255.255.0   UG    0      0        0 flannel.1
172.16.2.0      172.16.2.0      255.255.255.0   UG    0      0        0 flannel.1
172.16.3.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
192.168.10.0    0.0.0.0         255.255.255.0   U     100    0        0 eth0
```

### 3.VXLAN封装

各个宿主机VTEP设备之间组成一个二层网络，但是二层网络必须要知道目的的MAC地址，这个MAC地址在flanneld进程启动后，就会自动添加其他节点的ARP记录，可以通过`ip neighbo`命令查看，如下：`172.16.2.0`对应的mac地址为`22:56:a3:21:8b:8d`

```bash
[root@k8s-node02 ~]# ip neighbo show dev flannel.1
172.16.1.0 lladdr 4a:00:48:56:2b:49 PERMANENT
172.16.0.0 lladdr be:cc:92:39:62:18 PERMANENT
172.16.2.0 lladdr 22:56:a3:21:8b:8d PERMANENT
```

而`22:56:a3:21:8b:8d`就是目的容器所在宿主机的`flannel.1`虚拟网卡mac地址

```bash
[root@k8s-node01 ~]# ifconfig |grep "22:56:a3:21:8b:8d" -B 3
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 172.16.2.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::2056:a3ff:fe21:8b8d  prefixlen 64  scopeid 0x20<link>
        ether 22:56:a3:21:8b:8d  txqueuelen 0  (Ethernet)
```

### 4.二次封包

知道了目的地址的MAC地址，Linux内核就可以进行二层封包了。但是，对于宿主机网络来说这个二层帧并不能在宿主机二层网络里传输。所有接下来，Linux内核还要把这个数据帧进行进一步封装成为宿主机网络的一个普通数据帧，好让它载着内部数据帧，通过宿主机的eth0网卡来进行传输。数据格式如下图所示

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/202211151109106.png)

### 5.封装到udp包发出去

接着，VXLAN模块继续进行UDP封装，（要进行UDP封装，就要知道四元组信息：源IP、源端口、目的IP、目的端口）。首先是目的端口，Linux内核中默认为VXLAN分配的UDP监听端口为8472；然后是源端口，源端口是根据封装的内部数据帧做一个哈希值得到。

接下来是目的IP。Linux下的VXLAN设备都有一个转发表。通过下面的命令查看`flannel.1`的转发表，如下

```bash
[root@k8s-node03 ~]# bridge fdb show dev flannel.1
be:cc:92:39:62:18 dst 192.168.10.3 self permanent
02:96:f9:4b:d2:d7 dst 192.168.10.5 self permanent
22:56:a3:21:8b:8d dst 192.168.10.4 self permanent
```

它记录着，目的MAC地址为`22:56:a3:21:8b:8d`的数据帧封装后，应该发往哪个目的IP。根据上面的记录可以看出，UDP的目的IP应该为`192.168.10.4`。

最后是源IP，任何一个VXLAN设备创建时都会指定一个三层物理网络设备作为VTEP，这个物理网络设备的IP就是UDP的源IP。可以使用如下命令查看`flannel.1`的VTEP，查看后发现是eth0设备，所以源ip是`192.168.10.5`

```bash
[root@k8s-node02 ~]# ip -d link show flannel.1 
5: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 02:96:f9:4b:d2:d7 brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65535 
    vxlan id 1 local 192.168.10.5 dev eth0 srcport 0 0 dstport 8472 nolearning ttl auto ageing 300 udpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
```

### 6.数据包到达目的宿主机

接下来，就是宿主机和宿主机直接的通信了，数据包从k8s-node02的eth0网卡发出去，当数据包到达k8s-node01的8472端口后（实际上就是VXLAN模块），VXLAN模块就会比较这个VXLAN Header中的VNI和本机的VTEP（VXLAN Tunnel End Point，就是flannel.1）的VNI是否一致，然后比较Inner Ethernet Header中的目的MAC地址与本机的flannel.1是否一致，都一致后，则去掉数据包的VXLAN Header和Inner Ethernet Header，然后把数据包从flannel.1网卡进行发送。

然后，在k8s-node01节中上会有如下的路由（由flanneld维护），根据路由判断要把数据包发送到cni0网卡上。

```bash
[root@k8s-node01 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.10.254  0.0.0.0         UG    100    0        0 eth0
172.16.0.0      172.16.0.0      255.255.255.0   UG    0      0        0 flannel.1
172.16.1.0      172.16.1.0      255.255.255.0   UG    0      0        0 flannel.1
172.16.2.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
172.16.3.0      172.16.3.0      255.255.255.0   UG    0      0        0 flannel.1
192.168.10.0    0.0.0.0         255.255.255.0   U     100    0        0 eth0
```

### 7.tcpdump抓包在wireshark分析

进入容器ping另一个容器

```bash
|15:51:09|root@node01:[~]> kubectl exec -it nginx-d7b6f6c9c-49xl6 -- sh
/ # ping  172.16.2.30
PING 172.16.2.30 (172.16.2.30): 56 data bytes
64 bytes from 172.16.2.30: seq=0 ttl=62 time=0.510 ms
64 bytes from 172.16.2.30: seq=1 ttl=62 time=0.917 ms
```

抓取宿主机eth0网卡数据包

```bash
tcpdump  -i eth0 "udp and  host 192.168.10.5" -nnn  -w ping.cap
```

在wireshark分析，注意：默认情况下wireshark认为4789端口为vxlan协议，所以需要手动调整配置为8742端口解析vxlan协议

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/202211151554175.png)

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/202211151555270.png)

过滤容器ip

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/202211151556203.png)

打开某个报文，报文结构确实和分析流程一致

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/202211151557089.png)

### 8.VXLAN模式特点

1) 先进行二层帧封装，再通过宿主机网络封装，接封装也一样，所有会增加性能开销
2) 读宿主机网络要求较低，只要三层网络可达即可。

## 三.Host-gw工作模式原理

host-gw模式相比vxlan简单了许多， 直接添加路由，将目的主机当做网关，直接路由原始封包。 