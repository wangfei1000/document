# centos7 安装smokeping
**参考连接：**[力哥的博客:smokeping与centos7那些事 ](http://www.aclstack.com/454.html)
## 1、安装环境声明
    [root@node1 ~]# cat /etc/redhat-release 
	CentOS Linux release 7.2.1511 (Core) 
	[root@node1 ~]# uname -r
	3.10.0-327.el7.x86_64

	另外还需要关闭防火墙和selinux

## 2、安装依赖包
	安装系统基础的组件和相关的开发工具包
	yum groupinstall "Compatibility libraries" "Base" "Development tools" -y
	安装smokeping的依赖包
	yum install -y perl perl-Net-Telnet perl-Net-DNS perl-LDAP perl-libwww-perl \
	perl-IO-Socket-SSL perl-Socket6 perl-Time-HiRes perl-ExtUtils-MakeMaker rrdtool \
	rrdtool-perl curl  httpd httpd-devel gcc make  wget libxml2-devel libpng-devel \
	glib pango pango-devel freetype freetype-devel fontconfig cairo cairo-devel \
	libart_lgpl libart_lgpl-devel perl-CGI-SpeedyCGI perl-Sys-Syslog popt-devel \
	libidn-devel fping ntpdate

## 3、安装smokeping
	cd /server/tools/
	wget http://pkgs.fedoraproject.org/repo/pkgs/smokeping/smokeping-2.6.8.tar.gz/md5/14a968daab2d17a27d41600077e3e967/smokeping-2.6.8.tar.gz
	tar xvf smokeping-2.6.8.tar.gz
	cd smokeping-2.6.8
	./setup/build-perl-modules.sh /usr/local/smokeping/thirdparty
	./configure --prefix=/usr/local/smokeping
 	/usr/bin/gmake install  # 第一次安装时会提示报错，可以忽略再次执行一次此命令就可安装成功。
 	/usr/bin/gmake install

## 4、smokeping配置修改
	cd /usr/local/smokeping/
	mkdir cache data var
	touch /var/log/smokeping.log
	chown apache:apache cache data var
	chown apache:apache /var/log/smokeping.log
	chmod 600 /usr/local/smokeping/etc/smokeping_secrets.dist
	cd /usr/local/smokeping/htdocs
	mv smokeping.fcgi.dist smokeping.fcgi
	cd /usr/local/smokeping/etc
	mv config.dist config

## 5、apache配置修改
	修改主配置文件
	vim /etc/httpd/conf/httpd.conf

	<Directory "/var/www/html"> 修改为 <Directory "/usr/local/smokeping">

	增加somekping配置
	vim /etc/httpd/conf.d/somekping.conf

	Alias /cache "/usr/local/smokeping/cache/"
	Alias /cropper "/usr/local/smokeping/htdocs/cropper/"
	Alias /smokeping "/usr/local/smokeping/htdocs/smokeping.fcgi"
	<Directory "/usr/local/smokeping">
	AllowOverride None
	Options All
	AddHandler cgi-script .fcgi .cgi
	Order allow,deny  
	Allow from all  
	DirectoryIndex smokeping.fcgi
	</Directory>

	systemctl restart httpd
	至此smokeping搭建完毕，不过现在还不能正常使用，需要修改字符集添加对中文的支持。

##６、字符集修改
在配置文件/usr/local/smokeping/etc/config，Presentation 下添加charset = utf-8

	vim /usr/local/smokeping/etc/config
	*** Presentation ***
	charset = utf-8                

安装字体包

	yum -y install wqy-zenhei-fonts      
	vim /usr/local/smokeping/lib/Smokeping/Graphs.pm

	my $val = 0;
	for my $host (@hosts){
		my ($graphret,$xs,$ys) = RRDs::graph
		("dummy",
		'--start', $tasks[0][1],
		'--end', $tasks[0][2],
		'--font TITLE:20""',   # 增加这一行
		"DEF:maxping=$cfg->{General}{datadir}${host}.rrd:median:AVERAGE",
		'PRINT:maxping:MAX:%le' );
		my $ERROR = RRDs::error();

## 7、启动smokeping
	/usr/local/smokeping/bin/smokeping 