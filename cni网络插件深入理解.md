## cni概念

CNI是Container Network Interface的缩写，是一个用于连接容器和网络的接口标准。在容器化平台中，容器需要和其他容器或者主机之间进行通信，CNI提供了一个标准的接口和插件机制，可以让容器和网络之间的通信更加灵活和可扩展。

CNI可以通过插件的形式集成到各种容器编排和管理平台中，如Kubernetes、Docker Swarm、Mesos等。CNI插件可以根据不同的网络环境和需求，选择不同的网络实现方式，如VLAN、VXLAN、IPsec等。

总之，CNI为容器网络提供了一个统一的接口和插件机制，使得容器的网络配置和管理更加灵活和可扩展。

CNI在GitHub上有两个项目，https://github.com/containernetworking/cni是cni插件的编写规范和一个用Go语言编写的库，可以利用这个库实现自己的cni插件。https://github.com/containernetworking/plugins是一些已经实现的标准插件，本例将通过plugins中的bridge插件演示一个cni的工作原理。

## 下载cni plugins

```
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
```

将下载好的cni plugins解压至/opt/cni/bin目录下，没有/opt/cni/bin可以先创建

```bash
tar xf cni-plugins-linux-amd64-v1.2.0.tgz -C /opt/cni/bin/
```

## 编写演示bridge cni网络配置文件

将一下内容保存为`demo.conflist`，调用该cni网络将会创建一个名为demo的网络，同时会创建br0的网桥

```json
{
  "cniVersion": "1.0.0",
  "name": "demo",
  "type": "bridge",
  "bridge": "br0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.0.0.0/24",
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ]
  }
}
```

各个字段解释

- cniVersion：代表CNI规范所用的版本
- name：目标网络的名称。
- type：所用插件的类型。
- bridge和isGateWay：这两个都是和bridge插件相关的特定参数
  - bridge：创建的网桥的名称
  - isGateWay：网桥是否是网关，连接的容器使用这个网关
- ipMasq：为目标网段配置地址伪装，即SNAT
- ipam：IP地址管理，提供了一系列方法用于对IP和路由进行管理，标准的ipam插件有：host-local、dhcp、static等，本例使用host-local ipam插件，只要提供配置信息，ipam插件就会在目标网段分配ip地址，为了确保ip地址的唯一性，默认会将分配好的信息保存在`/var/lib/cni/networks`下
  - type：指定所用IPAM插件的名称
  - subnet：为目标网络分配网段，包括网络ID和子网掩码，以CIDR形式标记
  - routes：用于指定路由规则，插件会为我们在容器的路由表里生成相应的规则，dst为`0.0.0.0/0`则代表“任何网络”，表示数据包将通过默认网关发往任何网络
  - rangeStart：分配ip的起始位置
  - rangeEnd：分配ip的结束位置
  - gateway：网关的ip地址，本例即为br0网桥的ip地址，不指定即为第一个可用ip

## 使用该网络

创建演示网络命名空间demo

```bash
ip netns add demo
```

查看网络命名空间

```bash
ip netns ls
```

使用该演示网络，主要通过环境变量传递给cni插件，网络配置文件则通过stdin的方式传给cni插件

```bash
CNI_COMMAND=ADD \
CNI_CONTAINERID=demo \
CNI_NETNS=/var/run/netns/demo \
CNI_IFNAME=eth0 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/bridge < demo.conflist
```

参数解释

- CNI_COMMAND：告诉CNI插件要执行的命令，允许的命令有ADD，DEL，CHECK，VERSION
- CNI_CONTAINERID：将要加入目标网络的容器所对应的network namespace的ID
- CNI_NETNS：容器对应的network namespace在宿主机上的文件路径。可以在`/var/run/netns`目录下查看
- CNI_IFNAME：作为veth pair在容器一端的网络接口，我们所期望的名称
- CNI_PATH：cni插件所在的目录

返回内容

```json
{
    "cniVersion": "1.0.0",
    "interfaces": [
        {
            "name": "br0",
            "mac": "2e:92:07:37:1c:f4"
        },
        {
            "name": "veth3428432e",
            "mac": "06:d0:b9:ab:3b:e5"
        },
        {
            "name": "eth0",
            "mac": "da:f4:ab:c1:47:2f",
            "sandbox": "/var/run/netns/demo"
        }
    ],
    "ips": [
        {
            "interface": 2,
            "address": "10.0.0.2/24",
            "gateway": "10.0.0.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        }
    ],
    "dns": {}
```

查看宿主机网络

```bash
|16:42:01|root@node01:[~]> ifconfig  br0
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.1  netmask 255.255.255.0  broadcast 10.0.0.255
        inet6 fe80::2c92:7ff:fe37:1cf4  prefixlen 64  scopeid 0x20<link>
        ether 2e:92:07:37:1c:f4  txqueuelen 1000  (Ethernet)
        RX packets 10  bytes 640 (640.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9  bytes 854 (854.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
|16:42:03|root@node01:[~]> brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.2e9207371cf4	no		veth3428432e
docker0		8000.0242f928f42e	no		
```
进入演示网络命令空间，可用看到eth0网卡确实分配了，默认路由也指向了br0的地址

```bash
|16:39:15|root@node01:[~]> ip netns exec demo ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.2  netmask 255.255.255.0  broadcast 10.0.0.255
        inet6 fe80::d8f4:abff:fec1:472f  prefixlen 64  scopeid 0x20<link>
        ether da:f4:ab:c1:47:2f  txqueuelen 0  (Ethernet)
        RX packets 12  bytes 1136 (1.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10  bytes 752 (752.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
|16:39:17|root@node01:[~]> ip netns exec demo route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.1        0.0.0.0         UG    0      0        0 eth0
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

## 清理演示环境

删除该网络

```bash
CNI_COMMAND=DEL \
CNI_CONTAINERID=demo \
CNI_NETNS=/var/run/netns/demo \
CNI_IFNAME=eth0 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/bridge < demo.conflist
```

删除网络命名空间

```bash
ip netns del demo
```

## docker容器使用演示网络

docker默认使用自己的网络实现方案，这里通过创建一个网络为none的容器，并使用之前的网络配置文件，为该容器配置网络。

```bash
docker run -d --name web --network none nginx:alpine
```

查看容器id和网络命名空间

```bash
docker inspect web |egrep "SandboxKey|Id"
```

使用cni插件为该docker容器分配网络

```bash
CNI_COMMAND=ADD \
CNI_CONTAINERID=df1d22615bdba6054091a60d63799341b5db3aeac059fb82e5192c01b0b73245 \
CNI_NETNS=/var/run/docker/netns/fc18add5beb3 \
CNI_IFNAME=eth0 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/bridge < demo.conflist
```

查看容器ip地址

```bash
|16:57:01|root@node01:[~]> docker exec -it web ifconfig
eth0      Link encap:Ethernet  HWaddr BE:51:B5:7C:F9:D1  
          inet addr:10.0.0.3  Bcast:10.0.0.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:7 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:666 (666.0 B)  TX bytes:42 (42.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

清理网络

```bash
CNI_COMMAND=DEL \
CNI_CONTAINERID=df1d22615bdba6054091a60d63799341b5db3aeac059fb82e5192c01b0b73245 \
CNI_NETNS=/var/run/docker/netns/fc18add5beb3 \
CNI_IFNAME=eth0 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/bridge < demo.conflist
```

清理容器

```bash
docker rm -f web
```

