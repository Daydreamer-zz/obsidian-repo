## 1.Remi源介绍
Remi repository 是包含最新版本 PHP 和 MySQL 包的 Linux 源，由 Remi 提供维护。
## 2.配置repo
去https://mirrors.ustc.edu.cn/remi/选择合适的el版本的rpm文件
![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/202211071606865.png)
```bash
rpm -ivh https://mirrors.ustc.edu.cn/remi/enterprise/remi-release-8.rpm
```
## 3.替换为国内源
```bash
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://rpms.remirepo.net|baseurl=http://mirrors.ustc.edu.cn/remi|g' \
    -e 's|^baseurl=http://rpms.remirepo.net|baseurl=http://mirrors.ustc.edu.cn/remi|g' \
    -i  /etc/yum.repos.d/remi*
```
## 4.更新缓存
```bash
dnf makecache
```
## 5.查看
```bash
|16:07:57|root@node01:[~]> dnf module list php
Last metadata expiration check: 0:20:37 ago on Mon 07 Nov 2022 03:47:28 PM CST.
Rocky Linux 8 - AppStream
Name                                   Stream                                     Profiles                                                     Summary                                                 
php                                    7.2 [d]                                    common [d], devel, minimal                                   PHP scripting language                                  
php                                    7.3                                        common [d], devel, minimal                                   PHP scripting language                                  
php                                    7.4                                        common [d], devel, minimal                                   PHP scripting language                                  
php                                    8.0                                        common [d], devel, minimal                                   PHP scripting language                                  

Remi's Modular repository for Enterprise Linux 8 - x86_64
Name                                   Stream                                     Profiles                                                     Summary                                                 
php                                    remi-7.2                                   common [d], devel, minimal                                   PHP scripting language                                  
php                                    remi-7.3                                   common [d], devel, minimal                                   PHP scripting language                                  
php                                    remi-7.4                                   common [d], devel, minimal                                   PHP scripting language                                  
php                                    remi-8.0                                   common [d], devel, minimal                                   PHP scripting language                                  
php                                    remi-8.1                                   common [d], devel, minimal                                   PHP scripting language                                  
php                                    remi-8.2                                   common [d], devel, minimal                                   PHP scripting language                                  

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
```
