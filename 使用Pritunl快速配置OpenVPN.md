# 使用Pritunl快速配置OpenVPN

## 1.安装软件包

这里使用ubuntu1804为例演示配置

### 1.1配置源

添加mongodb源

```bash
wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -
```

在`/etc/apt/sources.list.d`下新建`moongodb-org.list`文件，添加如下内容

```bash
deb https://mirrors.tuna.tsinghua.edu.cn/mongodb/apt/ubuntu bionic/mongodb-org/4.2 multiverse
```

添加Pritunl源

```bash
sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb http://repo.pritunl.com/stable/apt bionic main
EOF
```

```bash
curl https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc | sudo apt-key add -
```

### 1.2 安装

```bash
apt update
```

```bash
apt -y install pritunl mongodb-org
```

```bash
systemctl enable mongod.service pritunl.service
```


## 2.修改配置并启动

默认情况下，客户端在连接上OpenVPN后，全部流量都会走VPN Server代理，需要修改代码，直允许内网网段走vpn

进入如下目录

```bash
cd /usr/lib/pritunl/usr/lib/python3.9/site-packages/pritunl/clients
```

修改`clients.py`

```bash
vim /usr/lib/pritunl/usr/lib/python3.9/site-packages/pritunl/clients/clients.py
```

将如下代码进行修改，在183行附近

```python
if self.server.is_route_all():
    client_conf += 'push "redirect-gateway def1"\n'
```

修改为

```python
if self.server.is_route_all():
    client_conf += '\n'
```

如下图

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230629235000.png)

删除该目录下的`__pycache__`

```bash
rm -rf /usr/lib/pritunl/usr/lib/python3.9/site-packages/pritunl/clients/__pycache__
```

重新启动Pritunl

```bash
systemctl restart pritunl.service
```

## 3.配置OpenVPN server

默认情况下pritunl监听在443端口，访问部署的主机的https 443端口即可进入pritunl后台，首次安装按提示进行初始化配置

### 3.1 初始化pritunl

默认情况下会自动获取出网ip为Public Address，后续生成的用户配置文件将会塞入这个ip，如果dnat和snat ip不同，应该手动进行修改。

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630002447.png)

### 3.2 添加组织

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230629235426.png)

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230629235452.png)

### 3.3 添加server并管理组织

添加server关联至上一步创建的组织

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230629235612.png)

这里vpn网段、Port按需修改，之后安全组或者防火墙需要放行这个UDP端口，DNS Server留空

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630012320.png)

Server关联组织

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630002807.png)

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630002822.png)

添加内网网段路由，这里内网服务器网段为`192.168.7.0/24`

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630002937.png)

`0.0.0.0/0`默认路由按需删除

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630003024.png)

启动Server

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630003113.png)

### 3.4 添加用户

添加的用户需关联之前创建的组织

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630003245.png)

用户密码可以为空

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630003312.png)

## 4.安装配置客户端并连接

### 4.1 下载客户端

进入openvpn官网，https://openvpn.net/client/client-connect-vpn-for-windows/

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630003554.png)

### 4.2 下载用户配置文件

点击添加的用户的下载按钮，将获取到当前用户名的连接配置文件压缩包，解压完配置到openvpn客户端即可连接

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630003635.png)

点击浏览文件，找到上边解压的openvpn用户连接文件

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230630003824.png)