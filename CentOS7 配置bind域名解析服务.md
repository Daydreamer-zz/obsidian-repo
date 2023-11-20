	以demo.com为例，bind主服务器ip为`192.168.10.7`，bind从服务器为`192.168.10.8`

## 1.安装bind服务

```bash
yum install -y bind bind-utils
```

## 2.修改bind主配置文件

修改`/etc/named.conf`配置文件

```ini
################
options {
	# 修改127.0.0.1为any
	listen-on port 53 { any; };
	# 修改localhost为any
	allow-query     { any; };   
	
	# 除了本地配置的域名，其他都转发dns查询到如下dns服务器
	recursion yes;
    forwarders {
      114.114.114.114;
      223.5.5.5;
    };
    forward only;

	# 禁用安全检查，否则dns查询会失败
    dnssec-enable no;
    dnssec-validation no;
};

```

## 3.主服务器添加正向解析

在`/etc/named.rfc1912.zones`末尾添加一个正向解析区域

```
zone "demo.com" IN {
	type master;  
	file "demo.com.zone";
	allow-transfer { 192.168.10.8; };
	
	
};
```

在`/var/named/demo.com.zone`添加正向解析库文件并添加解析记录

```
$TTL 3600
@	IN 	SOA	demo.com. admin.demo.com (
					1683340187 ; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum

	IN	NS	ns1
	IN	NS	ns2
@	IN	A	192.168.10.10
ns1	IN	A	192.168.10.7
ns2	IN	A	192.168.10.8
dev	IN	A	192.168.10.252
www	IN	A	192.168.10.10
mysql	IN	A	192.168.10.20
web	IN	CNAME	www
```

检测解析库文件配置是否正确

```bash
named-checkzone demo.com /var/named/demo.com.zone
```

## 4.主服务器添加反向解析

在`/etc/named.rfc1912.zones`末尾添加一个反向解析区域

```
zone "10.168.192.in-addr.arpa" IN {
	type master;
	file "10.168.192.zone";
	allow-transfer { 192.168.10.8; };
};
```

在`/var/named/10.168.192.zone`添加正向解析库文件并添加解析记录

```
$TTL 3600
@	IN 	SOA	demo.com. admin.demo.com (
					1648525821 ; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )
					
	IN	NS	ns1.shz.com.
	IN	NS	ns2.shz.com.
7	IN	PTR	ns1.shz.com.
8	IN	PTR	ns2.shz.com.
10	IN	PTR	www.shz.com.
10	IN	PTR	web.shz.com.
20	IN	PTR	mysql.shz.com.
252	IN	PTR	dev.shz.com.
```

检测解析库文件配置是否正确

```bash
named-checkzone 10.168.192.in-addr.arpa /var/named/10.168.192.zone
```

## 5.配置bind从服务器

修改`/etc/named.conf`配置文件

```
################
options {
	listen-on port 53 { any; };  # 修改127.0.0.1为any
	allow-query     { any; };   # 修改localhost为any
};

```

在`/etc/named.rfc1912.zones`末尾添加正向和反向解析区域

```
zone "demo.com" IN {
        type slave;
        file "slaves/demo.com.zone";
        masters { 192.168.10.7; };
};

zone "10.168.192.in-addr.arpa" IN {
	type slave;
	file "slaves/10.168.192.zone";
        masters { 192.168.10.7; };
};
```

## 6.启动主从服务器上的bind服务

启动服务

```bash
systemctl enable named.service --now
```

检查正向解析

```bash
### 主bind服务器
[root@bind01 ~]#  dig www.demo.com @192.168.10.7

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.demo.com @192.168.10.7
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63805
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.demo.com.			IN	A

;; ANSWER SECTION:
www.demo.com.		3600	IN	A	192.168.10.10

;; AUTHORITY SECTION:
demo.com.		3600	IN	NS	ns2.demo.com.
demo.com.		3600	IN	NS	ns1.demo.com.

;; ADDITIONAL SECTION:
ns1.demo.com.		3600	IN	A	192.168.10.7
ns2.demo.com.		3600	IN	A	192.168.10.8

;; Query time: 0 msec
;; SERVER: 192.168.10.7#53(192.168.10.7)
;; WHEN: Sat May 06 10:41:14 CST 2023
;; MSG SIZE  rcvd: 125
```

```bash
### 从bind服务器
[root@bind01 ~]#  dig www.demo.com @192.168.10.8

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.demo.com @192.168.10.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51510
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.demo.com.			IN	A

;; ANSWER SECTION:
www.demo.com.		3600	IN	A	192.168.10.10

;; AUTHORITY SECTION:
demo.com.		3600	IN	NS	ns1.demo.com.
demo.com.		3600	IN	NS	ns2.demo.com.

;; ADDITIONAL SECTION:
ns1.demo.com.		3600	IN	A	192.168.10.7
ns2.demo.com.		3600	IN	A	192.168.10.8

;; Query time: 1 msec
;; SERVER: 192.168.10.8#53(192.168.10.8)
;; WHEN: Sat May 06 10:41:49 CST 2023
;; MSG SIZE  rcvd: 125
```

检查反向解析

```bash
### 主bind服务器
[root@bind01 ~]# dig -x 192.168.10.10 @192.168.10.7

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> -x 192.168.10.10 @192.168.10.7
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25007
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;10.10.168.192.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
10.10.168.192.in-addr.arpa. 3600 IN	PTR	web.shz.com.
10.10.168.192.in-addr.arpa. 3600 IN	PTR	www.shz.com.

;; AUTHORITY SECTION:
10.168.192.in-addr.arpa. 3600	IN	NS	ns2.shz.com.
10.168.192.in-addr.arpa. 3600	IN	NS	ns1.shz.com.

;; Query time: 0 msec
;; SERVER: 192.168.10.7#53(192.168.10.7)
;; WHEN: Sat May 06 10:42:15 CST 2023
;; MSG SIZE  rcvd: 134
```

```bash
### 从bind服务器
[root@bind01 ~]# dig -x 192.168.10.10 @192.168.10.8

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> -x 192.168.10.10 @192.168.10.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1590
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;10.10.168.192.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
10.10.168.192.in-addr.arpa. 3600 IN	PTR	www.demo.com.

;; AUTHORITY SECTION:
10.168.192.in-addr.arpa. 3600	IN	NS	ns2.demo.com.
10.168.192.in-addr.arpa. 3600	IN	NS	ns1.demo.com.

;; ADDITIONAL SECTION:
ns1.demo.com.		3600	IN	A	192.168.10.7
ns2.demo.com.		3600	IN	A	192.168.10.8

;; Query time: 0 msec
;; SERVER: 192.168.10.8#53(192.168.10.8)
;; WHEN: Sat May 06 10:42:41 CST 2023
;; MSG SIZE  rcvd: 149
```

## 7.注意事项

在修改完任意解析库文件后，每次修改应该同时修改serial编号，随机即可，之后在执行重载，以确保从服务器可以接收到

修改配置

```bash
$TTL 3600
@	IN 	SOA	demo.com. admin.demo.com (
					1683340188 ; serial  # 添加完记录修改这里
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum

	IN	NS	ns1
	IN	NS	ns2
@	IN	A	192.168.10.10
ns1	IN	A	192.168.10.7
ns2	IN	A	192.168.10.8
dev	IN	A	192.168.10.252
www	IN	A	192.168.10.10
mysql	IN	A	192.168.10.20
web	IN	CNAME	www
new	IN	A	192.168.10.30  # 添加新的记录
```

重载

```bash
rndc reload
```



