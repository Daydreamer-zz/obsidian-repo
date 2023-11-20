Calico 维护的网络在默认是（Node-to-Node Mesh）全互联模式，Calico集群中的节点之间都会相互建立连接，用于路由交换。但是随着集群规模的扩大，mesh模式将形成一个巨大服务网格，连接数成倍增加。

这时就需要使用 Route Reflector（路由器反射）模式解决这个问题。

确定一个或多个Calico节点充当路由反射器，集中分发路由，让其他节点从这个RR节点获取路由信息。

具体步骤如下：
## 1.关闭 node-to-node BGP网格
获取集群ASN号，执行`calicoctl get nodes -o wide`，可以看到如下集群ASN号为64512
```bash
|17:55:41|root@node01:[~]> calicoctl get nodes -o wide
NAME           ASN       IPV4              IPV6   
k8s-master01   (64512)   192.168.10.3/24          
k8s-node01     (64512)   192.168.10.4/24          
k8s-node02     (64512)   192.168.10.5/24          
k8s-node03     (64512)   192.168.10.6/24
```
关闭nodeToNodeMesh模式，即：将nodeToNodeMeshEnabled设置为false
```bash
cat << EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false  
  asNumber: 64512
EOF
```
## 2.配置指定节点充当路由反射器
给需要当路由反射器的节点打标签，这里选择k8s-node01
```bash
kubectl label node k8s-node01 route-reflector=true
```

获取节点配置文件
```bash
calicoctl get node k8s-node01 -o yaml
```

修改节点配置文件，在spec.bpg下添加`routeReflectorClusterID: 244.0.0.1`，这个值可以任意设置
```yaml
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"k8s-node01","kubernetes.io/os":"linux","node.kubernetes.io/node":"","route-reflector":"true"}'
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: k8s-node01
    kubernetes.io/os: linux
    node.kubernetes.io/node: ""
    route-reflector: "true"
  name: k8s-node01
spec:
  addresses:
  - address: 192.168.10.4/24
    type: CalicoNodeIP
  - address: 192.168.10.4
    type: InternalIP
  bgp:
    ipv4Address: 192.168.10.4/24
    routeReflectorClusterID: 244.0.0.1
  orchRefs:
  - nodeName: k8s-node01
    orchestrator: k8s
status:
  podCIDRs:
  - 172.16.3.0/24
```
## 3.添加BGPPeer
```bash
cat <<EOF|calicoctl create -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: peer-with-route-reflectors
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
EOF
```
## 4.查看节点BGP连接状态
去路由反射器节点查看
```bash
[root@k8s-node01 ~]# calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+---------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+--------------+---------------+-------+----------+-------------+
| 192.168.10.3 | node specific | up    | 10:03:59 | Established |
| 192.168.10.5 | node specific | up    | 10:03:59 | Established |
| 192.168.10.6 | node specific | up    | 10:04:01 | Established |
+--------------+---------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

非路由反射器节点
```bash
[root@k8s-master01 ~]# calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+---------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+--------------+---------------+-------+----------+-------------+
| 192.168.10.4 | node specific | up    | 10:03:59 | Established |
+--------------+---------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

