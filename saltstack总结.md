# saltstack总结
[官方中文网站](http://www.saltstack.cn/kb/salt-intro-1/)

[官方英文网站](http://www.saltstack.com/)

## 一、 简介
**三大功能**

1. 远程执行
2. 配置管理 
3. 云管理

**四种运行方式**

1. local
2. master / minion c/s模式（常用）
3. syndic - (相当于zabbix proxy）
4. salt ssh

## 二、 安装
### 1. 环境声明
<pre>
系统版本和内核：
CentOS Linux release 7.2.1511 (Core) 
3.10.0-327.el7.x86_64

基础环境：
salt-master 10.0.0.204
salt-minion 10.0.0.203 
</pre>

### 2. 安装salt的repo库文件
<pre>
[root@salt-node4 ~]# yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-1.el7.noarch.rpm 
</pre>


### 3. 安装salt软件
#### master端
安装
<pre>
[root@salt-node4 ~]# yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-1.el7.noarch.rpm  
[root@salt-node4 ~]# yum -y install salt-master salt-minion
</pre>

启动

<pre>
[root@salt-node4 ~]# systemctl start salt-master
</pre>

#### minion端
安装
<pre>
[root@salt-node4 ~]# yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-1.el7.noarch.rpm 
[root@salt-node4 ~]# yum -y install salt-minion
</pre>


### 4. 修改salt-minion配置文件并启动
<pre>
vim /etc/salt/minion
...
16 master: 10.0.0.204
# 此id是salt-minion的唯一标识符，此处可不填写，启动salt后会采用系统的主机名作为id
101 #id:
...

启动salt-minion
[root@test-node3 ~]# systemctl start salt-minion
[root@test-node3 ~]# cat /etc/salt/minion_id 
test-node3.damaicha.org-203
</pre>

## 三 、远程执行
### 1. 在master上 ，查看master的公钥和minion发来的公钥
<pre>
[root@salt-node4 ~]# tree /etc/salt/pki
/etc/salt/pki
├── master
│   ├── master.pem
│   ├── master.pub # master的公钥
│   ├── minions
│   ├── minions_autosign
│   ├── minions_denied
│   ├── minions_pre # 这个目录下是minion客户端发来的公钥
│   │   ├── salt-node4.damaicha.org-204
│   │   └── test-node3.damaicha.org-203
│   └── minions_rejected
└── minion
    ├── minion.pem
    └── minion.pub
</pre>

### 2. master 接受minion发过来的公钥。
查看下key

<pre>
[root@salt-node4 ~]# salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
salt-node4.damaicha.org-204
test-node3.damaicha.org-203
Rejected Keys:
</pre>

 salt-key -a 接受指定主机发来的公钥

<pre>
[root@salt-node4 ~]# salt-key -a salt*
The following keys are going to be accepted:
Unaccepted Keys:
salt-node4.damaicha.org-204
Proceed? [n/Y] y
Key for minion salt-node4.damaicha.org-204 accepted.
[root@salt-node4 ~]#
</pre>

再次进行查看

<pre>
[root@salt-node4 ~]# tree /etc/salt/pki/
/etc/salt/pki/
├── master
│   ├── master.pem
│   ├── master.pub
│   ├── minions  # master在同意之后，会将minion的证书移动到minions目录下
│   │   ├── salt-node4.damaicha.org-204
│   │   └── test-node3.damaicha.org-203
│   ├── minions_autosign
│   ├── minions_denied
│   ├── minions_pre
│   └── minions_rejected
└── minion # 同时minion也会收到来自master的公钥
    ├── minion_master.pub
    ├── minion.pem
    └── minion.pub

7 directories, 7 files
[root@salt-node4 ~]#
</pre>

### 3.检查远程主机的状态
<pre>
[root@salt-node4 ~]# salt '*' test.ping
salt-node4.damaicha.org-204:
    True
test-node3.damaicha.org-203:
    True
</pre>

### 4.远程执行命令
<pre>
root@salt-node4 ~]# salt "*" cmd.run "hostname" 
test-node3.damaicha.org-203:
    test-node3.damaicha.org-203
salt-node4.damaicha.org-204:
    salt-node4.damaicha.org-204
</pre>

## 四、 配置管理
[官方文档](https://www.unixhot.com/docs/saltstack/index.html)

 state格式：YAML  后缀：.sls

YAML：三班斧

[YAML官方参考文档](https://www.unixhot.com/docs/saltstack/topics/yaml/index.html)

1、缩进
2个空格，不能使用tab
2、冒号   右边有一个空格
3、短横线  （后面都有一个空格）表示一个列表

### 1. 编写一个sls文件
master开启 file roots
<pre>
[root@salt-node4 ~]# vim /etc/salt/master481 file_roots:
481 file_roots:
482   base:
483     - /srv/salt
</pre>

重启salt-master
		
	[root@salt-node4 ~]# systemctl restart salt-master

新建salt目录

	[root@salt-node4 ~]# mkdir /srv/salt
	[root@salt-node4 ~]# mkdir /srv/salt/web

编写一个安装apache的sls模块文件
	
<pre>
[root@salt-node4 web]# cat apache.sls 
apache-install:
  pkg.installed:
    - names:
      - httpd
      - httpd-devel

apache-service:
  service.running:
    - name: httpd
    - enable: True
</pre>

执行这个自定义的模块

	[root@salt-node4 web]# salt "*" state.sls web.apache	

###2. 定义一个top file
定义一个top file 执行模块，放在base目录下。base目录设置在file root目录下。
他的作用是，可以定义哪些主机执行这个指定的模块。

<pre>
[root@salt-node4 salt]# pwd
/srv/salt
[root@salt-node4 salt]# cat top.sls 
base:
  'test-node3.damaicha.org-203':
    - web.apache
  'salt-node4.damaicha.org-204':
    - web.apache 
</pre>

执行这个top file文件

1、 需要查看salt执行这个操纵需要操作那些，然后在进行操作

	[root@salt-node4 salt]# salt '*' state.highstate test=True
2、 执行这状态模块

	[root@salt-node4 salt]# salt '*' state.highstate
	

## 五、saltstack和zeroMQ
1. 使用长连接的方式，当需要发布消息的时候直接发布即可。
2. 消息订阅-广播，一人说，多个人听。


4505端口和salt-minion建立一个长连接。
<pre>
[root@salt-node4 salt]# lsof -i :4505
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
salt-mini 3206 root   26u  IPv4  25721      0t0  TCP salt-node4.damaicha.org-204:33713->salt-node4.damaicha.org-204:4505 (ESTABLISHED)
salt-mast 3282 root   16u  IPv4  23094      0t0  TCP *:4505 (LISTEN)
</pre>

查看saltstack各个进程的名称。
需要安装python-setproctitle这个安装包

Python-setprocesstilte
<pre>
[root@salt-node4 salt]# ps aux|grep salt
root       3195  0.0  0.7 323484  3760 ?        Ss   Jan10   0:00 /usr/bin/python /usr/bin/salt-minion
root       3206  0.0  4.2 686432 20608 ?        Sl   Jan10   0:22 /usr/bin/python /usr/bin/salt-minion
root       3208  0.0  0.7 398568  3568 ?        S    Jan10   0:00 /usr/bin/python /usr/bin/salt-minion
root       3270  0.0  5.3 382168 25896 ?        Ss   Jan10   0:00 /usr/bin/python /usr/bin/salt-master
root       3281  0.0  2.0 309272  9860 ?        S    Jan10   0:00 /usr/bin/python /usr/bin/salt-master
root       3282  0.0  4.5 457940 21988 ?        Sl   Jan10   0:00 /usr/bin/python /usr/bin/salt-master
root       3285  0.0  4.8 392552 23304 ?        S    Jan10   0:00 /usr/bin/python /usr/bin/salt-master
root       3286  0.2  8.7 395512 42372 ?        S    Jan10   1:27 /usr/bin/python /usr/bin/salt-master
root       3375  3.3  6.1 1121104 30072 ?       Sl   Jan10  17:17 /usr/bin/python /usr/bin/salt-master
root       3376  0.0  5.3 382168 26076 ?        S    Jan10   0:00 /usr/bin/python /usr/bin/salt-master
root       3386  0.0  5.3 759024 25776 ?        Sl   Jan10   0:01 /usr/bin/python /usr/bin/salt-master
root       3406  0.0  6.6 619368 32284 ?        Sl   Jan10   0:01 /usr/bin/python /usr/bin/salt-master
root       3407  0.0  6.4 620716 31408 ?        Sl   Jan10   0:01 /usr/bin/python /usr/bin/salt-master
root       3410  0.0  6.5 619976 31904 ?        Sl   Jan10   0:01 /usr/bin/python /usr/bin/salt-master
root       3411  0.0  6.8 621428 33208 ?        Sl   Jan10   0:01 /usr/bin/python /usr/bin/salt-master
root       3412  0.0  6.6 619516 32136 ?        Sl   Jan10   0:01 /usr/bin/python /usr/bin/salt-master
root      32466  0.0  0.2 112648   976 pts/0    S+   07:20   0:00 grep --color=auto salt
[root@salt-node4 salt]# systemctl restart salt-master
[root@salt-node4 salt]# ps aux|grep salt             
root       3195  0.0  0.7 323484  3760 ?        Ss   Jan10   0:00 /usr/bin/python /usr/bin/salt-minion
root       3206  0.0  4.2 686432 20792 ?        Sl   Jan10   0:22 /usr/bin/python /usr/bin/salt-minion
root       3208  0.0  0.7 398568  3568 ?        S    Jan10   0:00 /usr/bin/python /usr/bin/salt-minion
root      32473 32.0  7.3 384236 35852 ?        Ss   07:20   0:00 /usr/bin/python /usr/bin/salt-master ProcessManager # 主进程
root      32484  0.0  3.8 311340 18644 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master MultiprocessingLoggingQueue
root      32485  0.0  5.5 460008 27028 ?        Sl   07:20   0:00 /usr/bin/python /usr/bin/salt-master ZeroMQPubServerChannel # zeromq 消息队列
root      32488  0.0  5.4 378080 26644 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master EventPublisher
root      32489 10.5  6.8 386060 33104 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master Maintenance
root      32543  1.0  6.3 1122300 30672 ?       Sl   07:20   0:00 /usr/bin/python /usr/bin/salt-master Reactor
root      32544  2.0  6.1 384236 30072 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master ReqServer_ProcessManager
root      32546  0.0  6.2 564484 30564 ?        Sl   07:20   0:00 /usr/bin/python /usr/bin/salt-master MWorkerQueue
root      32562 47.0  6.7 386492 32656 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master MWorker-0
root      32563 50.0  6.7 386496 32660 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master MWorker-1
root      32564 47.0  6.7 386500 32660 ?        R    07:20   0:00 /usr/bin/python /usr/bin/salt-master MWorker-2
root      32565 45.0  6.7 386496 32660 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master MWorker-3
root      32566 43.0  6.7 386504 32668 ?        S    07:20   0:00 /usr/bin/python /usr/bin/salt-master MWorker-4
root      32899  0.0  0.2 112648   976 pts/0    S+   07:20   0:00 grep --color=auto salt
[root@salt-node4 salt]# 
</pre>

## 六、数据系统grains
grains 静态数据 是key values型。

当Minion 启动时收集minion本地的相关信息 。 例如：操作系统版本，内核版本，cpu, 内存， 硬盘， 设备型号， 序列号....


作用：

1. 目标选择。
2. 资产管理，信息查询。
3. 配置管理中使用。
4. 定义位置：minion客户端。

###1.grains基础操作。

1、查看salt默认自带的grains所有的Key。

	[root@salt-node4 salt]# salt 'test*'  grains.ls
	test-node3.damaicha.org-203:
    - SSDs
    - biosreleasedate
    - biosversion
    - cpu_flags
    - cpu_model
    - cpuarch

2、查看所有的key对应的values

	[root@salt-node4 salt]# salt 'test*'  grains.items
	 ...
	 .....
	    uid:
        0
    username:
        root
    uuid:
        564df67d-68e6-39f0-4083-e6bcbcd3fb38
    virtual:
        VMWare
    zmqversion:
        4.1.4
        
        
3、查看指定的key对应的values
  
  	例如查看ip
	[root@salt-node4 salt]# salt '*' grains.item fqdn_ip4 
	salt-node4.damaicha.org-204:
	    ----------
	    fqdn_ip4:
	        - 10.0.0.204
	test-node3.damaicha.org-203:
	    ----------
	    fqdn_ip4:
	        - 10.0.0.203

4、根据grains来进行目标选择。

	[root@salt-node4 salt]# salt -G "fqdn_ip4:10.0.0.204" cmd.run "hostname"
	salt-node4.damaicha.org-204:
	    salt-node4.damaicha.org-204
	
	
### 2.自定义grains
1、修改客户端minion主配置文件。

	[root@test-node3 ~]# vim /etc/salt/minion
	118 grains:
	119   roles:
	120     - apache

2、重启minion。

	[root@test-node3 ~]# systemctl restart salt-minion

>如果不想在mininon上重启客户端可以这样操作，进行刷新grains操作。
	
	salt 'test*' saltutil.sync_grains	
	
	

3、master端进行测试。

	[root@salt-node4 salt]# salt -G "roles:apache" test.ping
	test-node3.damaicha.org-203:
   		 True
	[root@salt-node4 salt]# salt 'test-*' grains.item roles
	test-node3.damaicha.org-203:
    ----------
    roles:
        - apache

### 3.自定义一个grains文件
1、自定义grains文件，一般放在/etc/salt/grains文件里，salt启动时会自动的去加载。
	
	[root@test-node3 ~]# vim /etc/salt/grains 
	wangfei: heheda

2、刷新minion grains

	[root@salt-node4 salt]# salt "test*" saltutil.sync_grains 
	test-node3.damaicha.org-203:
	
3、master端测试
 
	[root@salt-node4 salt]# salt 'test-*' grains.item wangfei
	test-node3.damaicha.org-203:
    ----------
    wangfei:
        heheda

### 4.top file 结合grains使用

	[root@salt-node4 salt]# vim top.sls 
	base:
	  'test-node3.damaicha.org-203':
	    - web.apache
	  'salt-node4.damaicha.org-204':
	    - web.apache
	  'wangfei:heheda':
	    - match: grain
	    - web.apache
	    - 
	[root@salt-node4 salt]# salt '*'  state.highstate
	
### 5.自定义一个grains
1、新建_grains文件夹（必须放在/srv/salt/目录下。）

	[root@salt-node4 salt]# cd /srv/salt/
	[root@salt-node4 salt]# mkdir _grains

2、定义自定义的grains文件。

	[root@salt-node4 salt]# cd _grains/
	[root@salt-node4 _grains]# vim my_grains.py
	#!/bin/env python
	#-*- coding:utf-8 -*-
	
	def my_grains():
	    #初始化一个grains字典
	    grains = {}
	    #设置字典中的值 key-values
	    grains['iaas'] = 'openstack'
	    grains['edu'] = 'oldboyedu'
	    # 返回这个字典
	    return grains
	    
3、my_grains.py 推送到minion上去，此时已经同时刷新grains了。

	[root@salt-node4 _grains]# salt '*' saltutil.sync_grains 
	salt-node4.damaicha.org-204:
	    - grains.my_grains
	test-node3.damaicha.org-203:
	    - grains.my_grains
	 
4、minion客户端查看推送过来的grains文件。
>文件在/var/cache/salt/目录extmods目录下。

	[root@test-node3 ~]# tree /var/cache/salt/
	/var/cache/salt/
	└── minion
	    ├── accumulator
	    ├── extmods
	    │   └── grains
	    │       ├── my_grains.py
	    │       └── my_grains.pyc
	    ├── files
	    │   └── base
	    │       ├── _grains
	    │       │   └── my_grains.py  # here
	    │       ├── top.sls
	    │       └── web
	    │           └── apache.sls
	    ├── highstate.cache.p
	    ├── module_refresh
	    ├── pkg_refresh
	    ├── proc
	    └── sls.p
	
	9 directories, 9 files
	[root@test-node3 ~]#
	
5、master查看我定义的grains

	[root@salt-node4 _grains]# salt 'tes*' grains.item iaas 
	test-node3.damaicha.org-203:
	    ----------
	    iaas:
	        openstack
	[root@salt-node4 _grains]# salt 'tes*' grains.item edu
	test-node3.damaicha.org-203:
	    ----------
	    edu:
	        oldboyedu

### 6.定义的grains优先级

grains优先级：

	1. 系统自带的。
	2. grains文件里写的。
	3. minion配置文件里写的。
	4. 自己写的grains。

## 七、数据系统pillar

pillar动态，给特定的minion指定特定的数据。只用指定的minion才能看到自己的数据

>**在什么地方使用呢？**

	1. 定义一些敏感的数据时使用。
	2. 所有的变量都可以用pillar来进行定义。例如100台机器中有1台不一样，就可以用pillar来进行管理。
	3. 目标选择。-I
	4. 定义位置在master端。

1、查看pillar的值。
>默认是没有值的，因为pillar在配置文件里面默认是关闭的。

打开配置文件里面的Pillar

	[root@salt-node4 ~]# vim /etc/salt/master
	664 pillar_opts: True   # 是pillar_opts 这一行哦。

重启master 查看默认的pillar

	[root@salt-node4 ~]# systemctl restart salt-master
	[root@salt-node4 ~]# salt 'test-*' pillar.items 
	test-node3.damaicha.org-203:
	    ----------
	    master:
	        ----------
	        __role:
	            master
	        auth_mode:
	            1
	.....................
	看完之后，就关闭它。一会我们要自定义一个pillar

2、开启pillar_roots。

	[root@salt-node4 salt]# vim /etc/salt/master
	641 pillar_roots:
	642   base:
	643     - /srv/pillar
	
	重启master
	[root@salt-node4 ~]# systemctl restart salt-master
	

3、新建pillar文件夹。

	[root@salt-node4 ~]# mkdir /srv/pillar
	[root@salt-node4 ~]# tree /srv/
	/srv/
	├── pillar
	└── salt
	    ├── _grains
	    │   └── my_grains.py
	    ├── top.sls
	    └── web
	        └── apache.sls
	
	4 directories, 3 files

4、新建pillar的sls文件。
> jinjia模板必须结合top file才能使用。

	[root@salt-node4 ~]# cd /srv/pillar/
	[root@salt-node4 pillar]# mkdir web
	[root@salt-node4 pillar]# vim web/apache.sls # 这是一个jinjia模板
	{% if grains['os'] == 'CentOS' %}
	apache: httpd
	{% elif grains['os'] == 'Debian' %}
	apache: apache2
	{% endif %}	
	
5、新建pillar 的 topfile 文件。

	[root@salt-node4 pillar]# vim top.sls
	base:
	  'test-node3.damaicha.org-203':
	    - web.apache

6、master刷新pillar。

	[root@salt-node4 pillar]# salt '*' saltutil.refresh_pillar
	]test-node3.damaicha.org-203:
	    True
	salt-node4.damaicha.org-204:
	    True

7、master 测试。
	
	[root@salt-node4 pillar]# salt '*' pillar.items  # 查看所有的pillar      
	salt-node4.damaicha.org-204:
	    ----------
	test-node3.damaicha.org-203:
	    ----------
	    apache:
	        httpd
	
	[root@salt-node4 pillar]# salt '*' pillar.item apache # 查看指定的pillar
	test-node3.damaicha.org-203:
	    ----------
	    apache:
	        httpd
	salt-node4.damaicha.org-204:
	    ----------
	    apache:
	
匹配pillar

	[root@salt-node4 pillar]# salt -I "apache:httpd" cmd.run "hostname"
	test-node3.damaicha.org-203:
	    test-node3.damaicha.org-203
             
             
## 八、pillar和grains的区别
类型：

	pillar 动态的
	grains 静态的
	
数据收集方式:

	grains minion启动时收集
	pillar master自定义
	
应用场景：
	
	grains 目标查询、目标选择、配置管理
	pillar 目标选择、配置管理、敏感数据
	
定义位置：
	
	grains minion客户端
	pillar master服务端
	

## 九、远程执行 - 指定目标	

	salt '*' cmd.run "hostname"
	分解 
		命令：salt
		目标：*
		模块：cmd.run
		返回：hostname执行后就是返回结果。
	
1、 目标。

	有2种：
		1. 和minion id 有关的。
		2. 和minion id 无关的。

###minion id 有关的

>通配符
	
	例子：
	salt 'test-node3.damaicha.org-203' test.ping
	salt 'test-*' test.ping    
	salt 'test-node[0-9].damaicha.org-203' test.ping                  
	
>正则表达式 -E

	例子：
	salt -E '(test|salt)-node[0-9].damaicha.org-20[0-9]' test.ping  

>列表 -L

	例子：
	salt -L 'test-node3.damaicha.org-203,salt-node4.damaicha.org-204' test.ping
	
	

###minion id无关的

>pillar -I

	例子：
	salt -I "apache:httpd" cmd.run "hostname"


>grains -G

	例子：
	salt -G "wangfei:heheda" cmd.run "hostname"

>node -N
	
	1、定义好Node
		vim /etc/salt/master
			952 nodegroups:
			953   web: 'L@test-node3.damaicha.org-203,salt-node4.damaicha.org-204'、
	2、重启master
		systemctl restart salt-master
		
	3、测试
		[root@salt-node4 pillar]# salt -N "web" test.ping      
		salt-node4.damaicha.org-204:
		    True
		test-node3.damaicha.org-203:
		    True
>混合匹配 -C

[官方参考资料 - 混合匹配](https://www.unixhot.com/docs/saltstack/topics/targeting/compound.html)

**注意:**混合匹配也同样适用于topfile文件

	例子：	
		salt -C 'G@os:Debian and webser* or E@db.*' test.ping
		

	Letter	Match Type	Example	Alt Delimiter?
	G	Grains glob	G@os:Ubuntu	Yes
	E	PCRE Minion ID	E@web\d+\.(dev|qa|prod)\.loc	No
	P	Grains PCRE	P@os:(RedHat|Fedora|CentOS)	Yes
	L	List of minions	L@minion1.example.com,minion3.domain.com or bl*.domain.com	No
	I	Pillar glob	I@pdata:foobar	Yes
	J	Pillar PCRE	J@pdata:^(foo|bar)$	Yes
	S	Subnet/IP address	S@192.168.1.0/24 or S@192.168.1.100	No
	R	Range cluster
	

>批处理 -b

	百分比重启，先重启50台，再重启50台机器。
	10台10台的
	salt '*' -b 10 test.ping
	按照百分比来
	salt -G 'os:RedHat' --batch-size 25% apache.signal restart
	
#注意

**所有匹配目标的方式都可以用到top file里来指定目标。**

##十、远程执行 - 执行模块

[saltstack所有的模块官方连接](https://www.unixhot.com/docs/saltstack/ref/modules/all/index.html#all-salt-modules
)

saltstack所有的自带模块在/usr/lib/python2.7/site-packages/salt/modules里。

现在执行一个自带的模块。

获取所有的tcp连接
	
	[root@salt-node4 pillar]# salt 'test*' network.active_tcp
	test-node3.damaicha.org-203:
	    ----------
	    0:
	        ----------
	        local_addr:
	            ::
	        local_port:
	            80
	        remote_addr:
	            ::
	        remote_port:
	            0
	    1:
	        ----------
	        local_addr:
	            ::
	        local_port:
	            52113
	        remote_addr:
	            ::
	        remote_port:
	            0
	    2:
	        ----------
	        local_addr:
	            10.0.0.203
	        local_port:
	            56828
	        remote_addr:
	            10.0.0.204
	        remote_port:
	            4505
	
获取主机名

	[root@salt-node4 pillar]# salt 'test*' network.get_hostname
	test-node3.damaicha.org-203:
    	test-node3.damaicha.org-203
 
 查看ssh服务是否在运行
 
 [saltstack所有服务相关的服务](https://www.unixhot.com/docs/saltstack/ref/modules/all/salt.modules.service.html#module-salt.modules.service)
 
	[root@salt-node4 pillar]# salt 'test*' service.available sshd
	test-node3.damaicha.org-203:
	    True

查看所有正在运行的服务

	[root@salt-node4 pillar]# salt 'test*' service.get_all
	
复制文件cp
> 将master里/etc/hosts 复制到目标主机为/tmp/hehe

	[root@salt-node4 pillar]# salt-cp "*" /etc/hosts /tmp/wf
	salt-node4.damaicha.org-204:
	    ----------
	    /tmp/wf:
	        True
	test-node3.damaicha.org-203:
	    ----------
	    /tmp/wf:
	        True
	        
	        
查看top file文件在minion需要做哪些事情。
	        
	[root@salt-node4 pillar]# salt '*' state.show_top
	test-node3.damaicha.org-203:
	    ----------
	    base:
	        - web.apache
	        - web.apache
	salt-node4.damaicha.org-204:
	    ----------
	    base:
	        - web.apache

手动的安装一个软件

	[root@salt-node4 pillar]# salt "*" state.single pkg.installed name=lsof
	
##十一、saltstack远程执行-返回程序

[官方相关参考文档](https://www.unixhot.com/docs/saltstack/ref/returners/index.html)

**返回数据到mysql**

[官方相关参考文档 - return mysql](https://www.unixhot.com/docs/saltstack/ref/returners/all/salt.returners.mysql.html)

1、minion安装mysql的pyhon客户端。

	[root@salt-node4 ~]#salt -G "wangfei:heheda" state.single pkg.installed name=MySQL-python
	
2、mysql数据库新建对应的库和表。

新建数据库

	CREATE DATABASE  `salt`
	  DEFAULT CHARACTER SET utf8
	  DEFAULT COLLATE utf8_general_ci;
	
	USE `salt`;
	
创建对应的表

	DROP TABLE IF EXISTS `jids`;
	CREATE TABLE `jids` (
	  `jid` varchar(255) NOT NULL,
	  `load` mediumtext NOT NULL,
	  UNIQUE KEY `jid` (`jid`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;
	CREATE INDEX jid ON jids(jid) USING BTREE;
	
	--
	-- Table structure for table `salt_returns`
	--
	
	DROP TABLE IF EXISTS `salt_returns`;
	CREATE TABLE `salt_returns` (
	  `fun` varchar(50) NOT NULL,
	  `jid` varchar(255) NOT NULL,
	  `return` mediumtext NOT NULL,
	  `id` varchar(255) NOT NULL,
	  `success` varchar(10) NOT NULL,
	  `full_ret` mediumtext NOT NULL,
	  `alter_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
	  KEY `id` (`id`),
	  KEY `jid` (`jid`),
	  KEY `fun` (`fun`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;

	DROP TABLE IF EXISTS `salt_events`;
	CREATE TABLE `salt_events` (
	`id` BIGINT NOT NULL AUTO_INCREMENT,
	`tag` varchar(255) NOT NULL,
	`data` mediumtext NOT NULL,
	`alter_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
	`master_id` varchar(255) NOT NULL,
	PRIMARY KEY (`id`),
	KEY `tag` (`tag`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;
	
	
3、新建对应的管理用户。

	grant all on salt.* to salt@'%' identified by 'salt@pw';

4、minion客户端配置文件，添加如下，并且重启。

	vim /etc/salt/minion
	
	mysql.host: '10.0.0.204'
	mysql.user: 'salt'
	mysql.pass: 'salt@pw'
	mysql.db: 'salt'
	mysql.port: 3306
	
	systemctl restart salt-minion

5、master端执行命令

	[root@salt-node4 ~]# salt '*' cmd.run "hostname" --return mysql

6、此时登录数据库进行查看。

	MariaDB [salt]> select * from salt_returns\G
	*************************** 8. row ***************************
	       fun: cmd.run
	       jid: 20170221043152141415
	    return: "Filesystem               Size  Used Avail Use% Mounted on\n/       dev/mapper/centos-root   18G  2.0G   16G  12% /\ndevtmpfs                 227M     0  227M   0% /dev\ntmpfs                    237M   12K  237M   1% /dev/shm\ntmpfs                    237M  4.6M  233M   2% /run\ntmpfs                    
	............................
	  
	  
## 十二、saltstack远程执行-编写执行模块

1、存放位置。

	[root@salt-node4 ~]# mkdir /srv/salt/_modules
2、新建执行模块。（文件名就是模块名。）

	[root@salt-node4 ~]# vim /srv/salt_modules/my_disk.py
	[root@salt-node4 /srv/salt/_modules]# vim my_disk.py
	def list():
	    cmd = 'df -h'
	    ret = __salt__['cmd.run'](cmd)
	    return ret

3、刷新，加载这个自定义的模块。

	[root@salt-node4 ~]# salt '*' saltutil.sync_modules
	salt-node4.damaicha.org-204:
	    - modules.my_disk
	test-node3.damaicha.org-203:
	    - modules.my_disk
	
	刷新后，会将这个文件放在/var/cache/salt/minion/extmods/modules/my_disk.py
	[root@test-node3 srv]# tree /var/cache/salt
	/var/cache/salt/
	└── minion
	    ├── accumulator
	    ├── extmods
	    │   ├── grains
	    │   │   ├── my_grains.py
	    │   │   └── my_grains.pyc
	    │   └── modules
	    │       └── my_disk.py
	    ├── files
	    │   └── base
	    │       ├── _grains
	    │       │   └── my_grains.py
	    │       ├── _modules
	    │       │   └── my_disk.py
	    │       ├── top.sls
	    │       └── web
	    │           └── apache.sls
	    ├── highstate.cache.p
	    ├── module_refresh
	    ├── proc
	    └── sls.p	

4、服务端执行。
	
	[root@salt-node4 ~]# salt '*' my_disk.list
	test-node3.damaicha.org-203:
	    Filesystem               Size  Used Avail Use% Mounted on
	    /dev/mapper/centos-root   18G  2.0G   16G  12% /
	    devtmpfs                 227M     0  227M   0% /dev
	    tmpfs                    237M   12K  237M   1% /dev/shm
	    tmpfs                    237M  4.6M  233M   2% /run
	    tmpfs                    237M     0  237M   0% /sys/fs/cgroup
	    /dev/sda1                497M  125M  373M  26% /boot
	    tmpfs                     48M     0   48M   0% /run/user/0
	salt-node4.damaicha.org-204:
	    Filesystem               Size  Used Avail Use% Mounted on
	    /dev/mapper/centos-root   18G  2.0G   16G  12% /
	    devtmpfs                 227M     0  227M   0% /dev
	    tmpfs                    237M   28K  237M   1% /dev/shm
	    tmpfs                    237M  8.6M  229M   4% /run
	    tmpfs                    237M     0  237M   0% /sys/fs/cgroup
	    /dev/sda1                497M  125M  373M  26% /boot
	    tmpfs                     48M     0   48M   0% /run/user/0
		