## 1.前言
默认情况下，Calico IPAM 默认块大小为 /26。要从默认大小 /26 扩展，请降低blockSize（例如 /24）
由于在blockSize安装 Calico 后无法直接编辑该字段，因此最好在安装前更改 IP 池块大小，以尽量减少对 pod 连接的中断。
## 2.下载calicoctl
应该下载和k8s集群安装calico版本相同的calicoctl版本，去calico项目仓库release下载对应平台即可，需要确保当前节点有kubectl和kubeconfig文件，https://github.com/projectcalico/calico/releases
## 3.备份原默认calico ippool
```bash
calicoctl get ippool default-ipv4-ippool -o yaml > default-ipv4-ippool.yaml
```
```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-12-14T09:33:25Z"
  name: default-ipv4-ippool
  resourceVersion: "1363"
  uid: d1a6f51c-515b-450e-b714-bee6b6f28b33
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 172.16.0.0/16
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```
## 4.创建临时 ippool
创建临时ip池
```bash
cat <<EOF|calicoctl create -f -
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: temporary-pool
spec:
  cidr: 10.0.0.0/16
  ipipMode: Always
  natOutgoing: true
EOF
```
查看ip池
```bash
|17:41:24|root@node01:[~]> calicoctl get ippool -o wide
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   172.16.0.0/16   true   Never      Never       false      false              all()      
temporary-pool        10.0.0.0/16     true   Always     Never       false      false              all()
```
## 5.禁用原来ip池
执行禁用
```bash
calicoctl patch ippool default-ipv4-ippool -p '{"spec": {"disabled": true}}'
```
查看状态
```bash
|17:43:16|root@node01:[~]> calicoctl get ippool -o wide
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   172.16.0.0/16   true   Never      Never       true       false              all()      
temporary-pool        10.0.0.0/16     true   Always     Never       false      false              all()
```
## 6.删除原来已经分配ip的pod
<font color='red'>危险命令，注意一定要在刚刚安装完集群再操作</font>
```bash
kubectl delete pod -A --all
```
## 7.删除原来默认的ip pool
```bash
calicoctl delete ippool default-ipv4-ippool
```
## 8.修改原来的ippool文件
修改blockSize到24
```bash
cat <<EOF|calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 24
  cidr: 172.16.0.0/16
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
EOF
```
## 9.禁用临时ippool
```bash
calicoctl patch ippool temporary-pool -p '{"spec": {"disabled": true}}'
```
## 10.再次删除所有已经分配ip的pod
<font color='red'>危险命令，注意一定要在刚刚安装完集群再操作</font>
```bash
kubectl delete pod -A --all
```
## 11.删除临时ippool
```bash
calicoctl delete pool temporary-pool
```
## 12.查看k8s集群ip分配状态
```bash
|17:50:20|root@node01:[~]> calicoctl ipam show --show-blocks
+----------+----------------+-----------+------------+--------------+
| GROUPING |      CIDR      | IPS TOTAL | IPS IN USE |   IPS FREE   |
+----------+----------------+-----------+------------+--------------+
| IP Pool  | 172.16.0.0/16  |     65536 | 1 (0%)     | 65535 (100%) |
| Block    | 172.16.30.0/24 |       256 | 1 (0%)     | 255 (100%)   |
+----------+----------------+-----------+------------+--------------+
```
