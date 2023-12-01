## 1.安装postgresql repo
```bash
rpm -ivh https://mirrors.ustc.edu.cn/postgresql/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
## 2.修改为国内镜像
```bash
sed -i "s#download.postgresql.org/pub#mirrors.ustc.edu.cn/postgresql#g" /etc/yum.repos.d/pgdg-redhat-all.repo
```
## 3.启用powertools仓库
```bash
dnf config-manager --set-enabled powertools
```
## 4.重新生成dnf缓存
```bash
dnf makecache -y
```
## 5.禁用系统postgresql repo
```bash
dnf -qy module disable postgresql
```
## 6.列出可选版本
_前为postgis版本，_后为postgresql版本
```bash
[root@localhost ~]# dnf list|grep postgis3|awk -F '.' '{print $1}'|grep -v -
postgis30_11
postgis30_12
postgis30_13
postgis31_11
postgis31_12
postgis31_13
postgis31_14
postgis32_11
postgis32_12
postgis32_13
postgis32_14
postgis32_15
postgis33_11
postgis33_12
postgis33_13
postgis33_14
postgis33_15
postgis34_12
postgis34_13
postgis34_14
postgis34_15
postgis34_16
```
## 7.安装指定版本postgis
以postgis34_12为例安装
```bash
dnf install -y postgis34_12 postgis34_12-client
```
## 8.初始化postgis数据库并启动
初始化数据库
```bash
/usr/pgsql-12/bin/postgresql-12-setup initdb
```
启动Postgis
```bash
systemctl enable postgresql-12.service --now
```