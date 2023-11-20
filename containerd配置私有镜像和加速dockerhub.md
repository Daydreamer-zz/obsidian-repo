containerd 1.6以上版本在配置私有镜像和加速弃用了之前的配置方案
## 1.修改config.toml
修改/etc/containerd/config.toml，确保config_path值为/etc/containerd/certs.d
```toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
```
## 2.新建目录
创建/etc/containerd/certs.d目录
```bash
mkdir -p /etc/containerd/certs.d
```
## 3.配置dockerhub加速
新建目录
```bash
mkdir -p /etc/containerd/certs.d/docker.io
```
在/etc/containerd/certs.d/docker.io下创建文件hosts.toml
```toml
server = "https://docker.io"
  
[host."https://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]
```
## 4.配置私有镜像harbor
新建目录，以实际配置的私有域名为准，如这里私有镜像地址为harbor.node1.com，则需要新建这个域名的目录
```bash
mkdir -p /etc/containerd/certs.d/harbor.node1.com
```
在/etc/containerd/certs.d/harbor.node1.com创建hosts.toml
```toml
server = "https://harbor.node1.com"

[host."https://harbor.node1.com"]
  capabilities = ["pull", "resolve"]
  skip_verify = true  # 跳过证书验证
```
## 5.重启containerd
```bash
systemctl restart containerd.service
```