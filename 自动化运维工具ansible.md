## 1.安装

```
yum install -y ansible cowsay
```

## 2.配置

```
vim /etc/ansible/hosts
```

将要管理的主机加入配置文件，前提要做好ssh-key秘钥登录，这里不做描述

```
[nodes]
192.168.2.4
192.168.2.5
192.168.2.6
```

## 3.常用模块

- **ping**

  检测被控主机是否能ping通

  ```bash
  ansible nodes -m ping  #ansible后加主机可以是all、在hosts文件配置的标签如nodes、或者ip，-m指定模块
  ```

- **command**

  执行命令，只能执行简单的命令，无法解释特殊符号，如管道|,统配*等等，不指定模块默认是执行此模块

  ```bash
  ansible nodes -m command -a "ifconfig"  #-a后指定动作
  ```

- **shell**

  可以识别bash变量、管道、通配等

  ```bash
  ansible nodes -m shell -a "hostname >/tmp/hostname.txt"
  ```

- **copy**

  把管理机的文件复制到被控主机

  ```bash
  ansibe nodes -m copy -a "src=/scripts/lnmp.sh dest=/root"
  ```

  ```bash
  ansibe nodes -m copy -a "src=/scripts/lnmp.sh dest=/root/LNMP.sh owner=nobody group=nobody mode=700" 
  ```

  生成空文件

  ```bash
  ansible nodes -m copy -a "content="" dest=/root/a.txt"
  ```


- **script**

  相当于结合`shell`模块和`copy`模块，先把脚本传到服务器上再执行

  ```bash
  ansible nodes -m script -a "/scripts/lnmp.sh"
  ```

- **file**

  修改文件用户，组，权限，路径，创建目录或文件

  要指定path，state(directory|touch|link)

  ```bash
  ansible nodes -m file -a "path=/www state=directory"
  ```

- **yum**

  指定包名，版本state有：present,latest

  ```bash
  ansible nodes -m yum -a "name=nginx state=present"
  ```

- **cron**

  定时任务，相当于`vi /var/spool/cron/root`

  ```bash
  ansible nodes -m cron -a 'name="backup etc" minute=00 hour=00 job="tar zcf /tmp/backup.tar.gz /backup/* >/dev/null 2>&1" state=present'
  ```

  删除某个定时任务，指定state为adsent即可，必须制定name

  ```bash
  ansible nodes -m cron -a 'name="backup etc" state=absent'
  ```

## 3.playbook剧本

- 示例play1.yml

  ```yaml
  ---
  - hosts: nodes
    remote_user: root
    tasks:
      - name: upload cache
        yum:
          update_cache: yes
      - name: install redis
        yum:
          name: redis
          state: present
      - name: copy redis conf
        tags: configredis
        copy:
          src: /root/redis.con
          dest: /etc/redis.conf
          owner: redis
          mode: 0640
        notify: restart redis
      - name: start redis
        service:
          name: redis
          state: started
    handlers:
      - name: restart redis
        service:
          name: redis
          state: restarted
  ```

  执行该playbook

  ```bash
  ansible-playbook -C /etc/ansible/play1.yml  #dry run
  ```

  ```bash
  ansible-playbook /etc/ansible/play1.yml
  ```

  执行指定tag任务

  ```bash
  ansible-playbook -t configredis play1.yml
  ```
## 4.absible变量

- ansible组变量定义格式

  通过定义在inventory主机清单中

  ```ini
  [nodes]
  #某个主机拥有的变量
  node01 bind_host=127.0.0.1
  node02 bind_host=0.0.0.0
  
  #noeds组中拥有该变量
  [nodes:vars]
  http_port=8080
  ```

  playbook中使用

  ```yaml
  ---
  - hosts: nodes
    tasks:
      - name: Shwo port
        debug:
          msg: "The group vars is {{ http_port }}"
  ```

- vars定义变量

  在playbook里使用变量，使用vars定义好后，用连个花括号表示引用`{{}}`

  ```yaml
  ---
    - hosts: nodes
      vars:
        file: shz.txt
        dir: /root/
      tasks:
        - name: touch file
          file: path={{dir}}/{{file}} state=touch
  ```
  
- 使用Facts中的变量

  使用setup模块查看可用变量(可用-a参数过滤)

  ```bash
  ansible node01 -m setup -a "filter=*env*"
  ```

  playbook中使用

  ```yaml
  ---
  - hosts: nodes
    remote_user: root
    tasks:
      - name: save env file
        copy:
          content: "{{ ansible_env }}"  
          dest: /root/ansible.env
          mode: 644
  ```

- 向playbook中传递变量(command line变量)

  ```yaml
  ---
  # playbook中使用变量
  - hosts: nodes
    remote_user: root
    tasks:
      - name: install {{ pkgname }}
        yum:
          name: "{{ pkgname }}"
          state: present
  ```

  执行

  ```bash
  ansible-playbook -e pkgname=nginx play3.yml
  ```
  
- 注册系统命令返回值作为变量

  ```yaml
  ---
    - hosts: nodes
      tasks:
        - name: get ip address
          shell: hostname -I
          register: ip
        - name: print ip var to file
          shell: echo {{ip.stdout}} >/tmp/ip.txt
  ```

  如下实例一个打包备份配置文件的playbook

  ```yaml
  ---
    - hosts: nodes
      tasks:
        - name: get ip
          shell: hostname -I
          register: ip
        - name: get date
          shell: date +%F
          register: date
        - name: mkdir 
          file: path=/backup/{{ip.stdout}} state=directory
        - name: tar
          shell: tar zcf /backup/etc-{{ip.stdout}}-{{date.stdout}}.tar.gz /etc/*
  ```

- 调试变量

  debug模块：msg={{xxx}}

  ```yaml
  ---
    - hosts: nodes
      tasks:
        - name: get ip
          shell: hostname -I
          register: ip
        - name: debug test
          debug: msg={{ip}}
  ```

## 5.ansible模板

使用redis.conf配置文件为例，修改redis-server监听本机ip地址

```bash
cp /etc/redis.conf /root/redis.conf.j2
```

修改redis.conf.j2中的bind为模板变量

```ini
'''
# IF YOU ARE SURE YOU WANT YOUR INSTANCE TO LISTEN TO ALL THE INTERFACES
# JUST COMMENT THE FOLLOWING LINE.
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bind {{ ansible_eth0.ipv4.address }}
'''
```

测试渲染模板playbook

```yaml
---
- hosts: nodes
  remote_user: root
  gather_facts: true
  tasks:
    - name: generate redis.conf
      template:
        src: /root/redis.conf.j2
        dest: /etc/redis.conf
      notify: restart redis-server
  handlers:
    - name: restart redis-server
      service:
        name: redis
        state: restarted
```

## 6.ansible循环和判断

- 循环

  列表定义

  ```yaml
  ---
    - hosts: nodes
      tasks:
        - name: show ip
          shell: echo 192.168.2.{{item}} >/tmp/test1.txt
          with_items:
            - 4
            - 5
            - 6
  ```

  字典定义

  ```yaml
  ---
  - hosts: nodes
    remote_user: root
    tasks:
      - name: add some group
        group:
          name: "{{ item }}"
          state: present
        with_items:
          - group11
          - group12
          - group13
      - name: add some user
        user:
          name: "{{ item.name }}"
          group: "{{ item.group }}"
          state: present
        with_items:
          - { name: "user11", group: "group11" }
          - { name: "user12", group: "group12" }
          - { name: "user13", group: "group13" }
  ```

- 判断

  条件，when指定主机名 `ansible_hostname`叫做ansible内置变量

  ```yaml
  ---
    - hosts: nodes
      tasks:
        - name: install nfs
          yum: name=nfs-utils,rpcbind state=present
          when: ( ansible_hostname == "node3" )
  ```

## 7.ansible roles

### 目录结构

角色集合：

```bsh
.
└── demo
    ├── mysql
    ├── nginx
    └── php
```

每个角色

```bash
demo/nginx
├── default  #设定默认变量是使用此目录的main.yml
├── files    #存放由copy或script等模块调用的文件
├── handlers #至少应该包含一个名为main.yml的文件，其它通过include包含
├── meta     #至少应该包含一个名为main.yml的文件，定义当前角色的特殊设定及依赖关系，其它通过include包含
├── tasks    #至少应该包含一个名为main.yml的文件，其它通过include包含
├── templates #template模块查找所需要模板文件的目录
└── vars     #至少应该包含一个名为main.yml的文件，其它通过include包含
```

## 8.ansible 开启ssh长连接

编辑`/etc/ansible/ansible.cfg`，将ssh_args注释打开，`ControlPersist`参数加入连接时间，以86400s(1d)为例

```ini
[ssh_connection]

# ssh arguments to use
# Leaving off ControlPersist will result in poor performance, so use
# paramiko on older platforms rather than removing it, -C controls compression use
ssh_args = -C -o ControlMaster=auto -o ControlPersist=86400s
```

## 9.ansible facts缓存

ansible-playbook默认情况下每次playbook执行都会执行gathering facts操作，可以将facts结果缓存到文件，或者redis中，**注意开启facts缓存需要将gathering设置为smart**

开启redis缓存还需要安装python的redis模块`pip install redis`

编辑`/etc/ansible/ansible.cfg`

```ini
[defaults]

....
# plays will gather facts by default, which contain information about
# the remote system.
#
# smart - gather by default, but don't regather if already gathered
# implicit - gather by default, turn off with gather_facts: False
# explicit - do not gather by default, must say gather_facts: True
gathering = smart
....

# redis缓存
fact_caching = redis
fact_caching_timeout = 86400
fact_caching_connection = 127.0.0.1:6379

# json文件缓存
#fact_caching = jsonfile
#fact_caching_timeout = 86400
#fact_caching_connection = /dev/shm/ansible_facts_cache/
```

## 10.ansible异步和轮询

默认情况下playbook中的任务执行时会一直保持连接,直到该任务在每个节点都执行完毕.有时这是不必要的,比如有些操作运行时间比SSH超时时间还要长.

解决该问题最简单的方式是一起执行它们,然后轮询直到任务执行完毕.

你也可以对执行时间非常长（有可能遭遇超时）的操作使用异步模式.

adhook下执行异步任务

```bash
ansible node -m shell -a "sleep 10;echo '111'" -B 10 -P 0 
# -B指定后台运行时间，-P指定轮询获取任务结果时间间隔,执行后返回任务id(jid)，通过这个jib可以获取任务执行结果

k8s-node02 | CHANGED => {
    "ansible_job_id": "851635117187.32539",
    "changed": true,
    "finished": 0,
    "results_file": "/root/.ansible_async/851635117187.32539",
    "started": 1
}
k8s-node01 | CHANGED => {
    "ansible_job_id": "410269696145.21043",
    "changed": true,
    "finished": 0,
    "results_file": "/root/.ansible_async/410269696145.21043",
    "started": 1
}
```

获取异步任务结果

```bash
ansible node -m async_status -a "jid=851635117187.32539"

k8s-node02 | CHANGED => {
    "ansible_job_id": "851635117187.32539",
    "changed": true,
    "cmd": "sleep 10;echo '111'",
    "delta": "0:00:10.005124",
    "end": "2022-01-14 15:43:08.680929",
    "finished": 1,
    "rc": 0,
    "start": "2022-01-14 15:42:58.675805",
    "stderr": "",
    "stderr_lines": [],
    "stdout": "111",
    "stdout_lines": [
        "111"
    ]
}

```

在playbook中使用

```yaml
---
- name: this is a async job demo
  hosts: node
  tasks:
  - name: async job demo 
    shell: sleep 10;hostname
    async: 11
    poll: 0
    register: job
  - name: show async job demo  id
    debug:
      msg: "The demo job id is {{ job.ansible_job_id }}"
```

## 11.动态inventory

 对于主机数量较多的情况下，单独的inventory文件无法维护，可以使用动态inventory脚本从cmdb或者api中获取主机信息，这里以mysql方式实现一个动态inventory脚本

```python
#!/usr/bin/env python3
import json
import pymysql
import argparse
from itertools import groupby
from operator import itemgetter
from collections import defaultdict


def query_res(host, port, user, password, db, target_host=False):
    conn = pymysql.connect(host=host, user=user, passwd=password, port=port, db=db)
    cursor = conn.cursor(pymysql.cursors.DictCursor)
    if not target_host:
        sql = """
        SELECT
            `hosts`.ip,
            groups.group_name,
            groups.ssh_port,
            groups.ssh_user,
            groups.ssh_password
        FROM
            `hosts`
        LEFT JOIN groups ON `hosts`.group_id = groups.id
        """
        cursor.execute(sql)
    elif target_host:
        sql = f"""
        SELECT
	        `hosts`.ip,
	        groups.group_name,
	        groups.ssh_port,
	        groups.ssh_user,
	        groups.ssh_password 
        FROM
	        `hosts`
	    LEFT JOIN groups ON `hosts`.group_id = groups.id 
            WHERE
	        `hosts`.ip = "{target_host}" OR `hosts`.hostname = "{target_host}"
        """
        cursor.execute(sql)
    return conn, cursor


def process_group_hosts(sorted_res, target_dict):
    for group_name, hosts in groupby(sorted_res, key=itemgetter("group_name")):
        target_dict[group_name]['hosts'] = [i['ip'] for i in hosts]


def process_group_vars(sorted_res, target_dict):
    for group_name, hosts in groupby(sorted_res, key=itemgetter("group_name")):
        vars_dic = {}
        for group_var in hosts:
            vars_dic['ansible_ssh_port'] = group_var['ssh_port']
            vars_dic['ansible_ssh_user'] = group_var['ssh_user']
            vars_dic['ansible_ssh_pass'] = group_var['ssh_password']
        target_dict[group_name]["vars"] = vars_dic


if __name__ == '__main__':
    tree = lambda: defaultdict(tree)
    ansible_dict = tree()

    parser = argparse.ArgumentParser()
    parser.add_argument("-l", "--list", help="get hosts list vars", action="store_true")
    parser.add_argument("-H", "--host", help="get a host vars")
    args = parser.parse_args()

    if args.list:
        conn, cursor = query_res("127.0.0.1", 3306, "root", "199747", "cmdb")
        res = cursor.fetchall()
        cursor.close()
        conn.close()
        ansible_dict['all'] = [i['ip'] for i in res]
        res.sort(key=itemgetter('group_name'))
        process_group_hosts(res, ansible_dict)
        process_group_vars(res, ansible_dict)
        ansible_dict = dict(ansible_dict)
        print(json.dumps(ansible_dict))
    elif args.host:
        try:
            conn, cursor = query_res("127.0.0.1", 3306, "root", "199747", "cmdb", args.host)
            res = cursor.fetchone()
            cursor.close()
            conn.close()
            del res["group_name"]
            res["ansible_host"] = res.pop("ip")
            res["ansible_port"] = res.pop("ssh_port")
            res["ansible_user"] = res.pop("ssh_user")
            res["ansible_password"] = res.pop("ssh_password")
        except TypeError:
            res = {}
        print(json.dumps(res))
```

## 12.使用回调插件

回调插件可以在响应事件时向Ansible添加新的行为。默认情况下,回调插件可以控制你在运行命令行程序时看到的大部分输出,但也可以用来添加额外的输出,与其他工具集成,并将事件marshall到存储后端

### 12.1 修改默认回调插件

同时只能有一个回调插件作为主要管理者，用于输出到屏幕。

如果需要替换，应该在这个插件中修改`CALLBACK_TYPE = stdout`，之后在`ansible.cfg`中配置stdout_callback

```ini
stdout_callback = json  # 以json格式输出结果
```

也可以使用自定义回调插件:

```ini
stdout_callback = mycallbak
```

默认情况下这仅对playbook生效，如果想让ad-hoc方式生效应该在ansible.cfg中做如下设置：

```ini
[defaults]
bin_ansible_callbacks = True
```

### 12.2 启用其他内置回调插件

大部分情况下，无论是内置的回调插件还是自定义回调插件，都需要在ansible.cfg中添加到白名单中，从而才能启用

```ini
callback_whitelist = timer,mail,profile_roles,custom_callback
```

- `timer`可以计算整个playbook的时间
- `mail`可以实现发送邮件的功能
- `profile_roles`在执行中添加用时时间
- `custom_callback`是自定义的插件

### 12.3 获取帮助

- 如下命令可以查看当前可用的回调插件列表

  ```bash
  ansible-doc -t callback -l
  ```

- 如下命令可以查看具体的回调插件的帮助文档

  ```bash
  ansible-doc -t callback <callback plugin name>
  ```


### 12.4 回调插件类型

回调插件类型在回调插件类中定义

```python
class CallbackModule(CallbackBase):
    CALLBACK_TYPE = 'notification'
```

不同的回调类型对于playbook的输出有不一样的效果

- `stdout`标准输出类型，用在回调的主管理者
- `aggregate`聚合类型，把此类型插件处理的结果和`stdout`类型插件合并一起输出到标准输出。例如：`timer`
- `notification`通知类型，不参与标准输出，也不影响标准输出插件的正常输出，只会 把执行playbook的返回值写入指定的媒介中。例如：`log_plays`,`mail`。假如自定义把执行playbook的结果输出到数据库中就可以使用此类型。

### 12.5 把返回结果输出到日志中

内置的回调插件`log_plays`会将playbook的返回信息输出到`/var/log/ansible/hosts`目录中。

可以在`ansible.cfg`中配置指定的目录，使用`log_folder`

例如：把日志存储到`/tmp/ansible/hosts/`目录下，可以在`ansible.cfg`文件的最后添加如下配置，**还需要将log_plays加入到插件白名单配置项中**

```ini
[callback_log_plays]
log_folder = /tmp/ansible/hosts/
```

## 13.自定义ansible回调插件

将ansble-playbook执行任务写入mysql数据库

### 13.1建立表结构

```sql
CREATE TABLE `playsresult` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user` varchar(16) NOT NULL,
  `host` varchar(32) NOT NULL,
  `category` varchar(11) NOT NULL,
  `result` text,
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 13.2 编写插件

```python
from __future__ import (absolute_import, division, print_function)
__metaclass__ = type


DOCUMENTATION = '''
    callback: mysql_plays
    type: notification
    short_description: write playbook output to mysql database
    version_added: historical
    description:
      - This callback writes playbook output to mysql database
    requirements:
     - Whitelist in configuration
     - A mysql server that can be connected and insert
    options:
      mysql_host:
        version_added: '2.9'
        default: localhost
        description: The mysql server ip address
        env:
          - name: ANSIBLE_MYSQL_HOST
        ini:
          - section: callback_mysql_plays
            key: mysql_host
      mysql_username:
        version_added: '2.9'
        default: root
        description: The mysql username
        env:
          - name: ANSIBLE_MYSQL_USERNAME
        ini:
          - section: callback_mysql_plays
            key: mysql_username
      mysql_password:
        version_added: '2.9'
        default: 123456
        description: The mysql password
        env:
          - name: ANSIBLE_MYSQL_PASSWORD
        ini:
          - section: callback_mysql_plays
            key: mysql_password
      mysql_port:
        version_added: '2.9'
        default: 3306
        description: The mysql server port
        env:
          - name: ANSIBLE_MYSQL_PORT
        ini:
          - section: callback_mysql_plays
            key: mysql_port
      mysql_database:
        version_added: '2.9'
        default: ansible
        description: The mysql database
        env:
          - name: ANSIBLE_MYSQL_DATABASE
        ini:
          - section: callback_mysql_plays
            key: mysql_database
'''

import getpass
import json
from ansible.errors import AnsibleError, to_native
from ansible.module_utils.common._collections_compat import MutableMapping
from ansible.parsing.ajson import AnsibleJSONEncoder
from ansible.plugins.callback import CallbackBase

try:
    import pymysql as dbDriver
    pwd = "password"
    Db = "database"
except ImportError:
    try:
        import MySQLdb as dbDriver
        pwd = "passwd"
        Db = "db"
    except ImportError:
        raise AnsibleError("No module pymysql or MySQLdb found!!\nUse pip install pymysql or MySQLdb")


# NOTE: in Ansible 1.2 or later general logging is available without
# this plugin, just set ANSIBLE_LOG_PATH as an environment variable
# or log_path in the DEFAULTS section of your ansible configuration
# file.  This callback is an example of per hosts logging for those
# that want it.


class CallbackModule(CallbackBase):
    """
    logs playbook results, per host, in /var/log/ansible/hosts
    """
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'mysql_plays'
    CALLBACK_NEEDS_WHITELIST = True

    TIME_FORMAT = "%b %d %Y %H:%M:%S"
    MSG_FORMAT = "%(now)s - %(category)s - %(data)s\n\n"

    def __init__(self):

        super(CallbackModule, self).__init__()

    def set_options(self, task_keys=None, var_options=None, direct=None):
        super(CallbackModule, self).set_options(task_keys=task_keys, var_options=var_options, direct=direct)
        self.mysql_host = self.get_option("mysql_host")
        self.mysql_port = self.get_option("mysql_port")
        self.mysql_username = self.get_option("mysql_username")
        self.mysql_password = self.get_option("mysql_password")
        self.mysql_database = self.get_option("mysql_database")
        self.user = getpass.getuser()

    def _mysql(self):
        db_conn = {
            "host": self.mysql_host,
            "port": int(self.mysql_port),
            "user": self.mysql_username,
            pwd: self.mysql_password,
            Db: self.mysql_database
        }
        db = dbDriver.connect(**db_conn)
        cursor = db.cursor()
        return db, cursor

    def logmysql(self, host, category, data):
        if isinstance(data, MutableMapping):
            if '_ansible_verbose_override' in data:
                # avoid logging extraneous data
                data = 'omitted'
            else:
                data = data.copy()
                data = json.dumps(data, cls=AnsibleJSONEncoder)

        sql = """INSERT INTO `playsresult` (`user`, `host`, `category`, `result`) VALUES (%s, %s, %s, %s);"""
        try:
            db, cursor = self._mysql()
            cursor.execute(sql, (self.user, host, category, data))
            db.commit()
            cursor.close()
            db.close()
        except Exception as e:
            raise AnsibleError("%s" % to_native(e))

    def runner_on_failed(self, host, res, ignore_errors=False):
        self.logmysql(host, 'FAILED', res)

    def runner_on_ok(self, host, res):
        self.logmysql(host, 'OK', res)

    def runner_on_skipped(self, host, item=None):
        self.logmysql(host, 'SKIPPED', '...')

    def runner_on_unreachable(self, host, res):
        self.logmysql(host, 'UNREACHABLE', res)

    def runner_on_async_failed(self, host, res, jid):
        self.logmysql(host, 'ASYNC_FAILED', res)

    def playbook_on_import_for_host(self, host, imported_file):
        self.logmysql(host, 'IMPORTED', imported_file)

    def playbook_on_not_import_for_host(self, host, missing_file):
        self.logmysql(host, 'NOTIMPORTED', missing_file)
```

### 13.3 编辑ansible配置文件

```ini
callback_whitelist = profile_roles,log_plays,mysql_plays


[callback_mysql_plays]
mysql_host = 127.0.0.1
mysql_port = 3306
mysql_username = root
mysql_password = 123456
mysql_database = cmdb
```

