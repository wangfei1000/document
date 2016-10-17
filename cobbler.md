# cobbler部署
## 环境声明

- 本次实验环境为如下
<pre>
[root@study2 tools]# cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@study2 tools]# uname -r
3.10.0-327.36.1.el7.x86_64
</pre>

- 关闭防火墙和selinux

<pre>
关闭selinux
[root@study2 tools]# sed -i 's@^SELINUX=.*@SELINUX=disabled@g' /etc/selinux/config 
[root@study2 tools]# setenforce 0
关闭iptables
[root@study2 tools]# iptables -F
[root@study2 tools]# iptables -X
[root@study2 tools]# iptables -Z
[root@study2 tools]# systemctl stop firewalld
</pre>

- cobbler服务器地址为10.0.0.201

## cobbler安装
- 添加使用阿里的yum源。

<pre>
[root@study2 ~]# cd /server/tools/
[root@study2 tools]# wget  http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
[root@study2 tools]# rpm -vih epel-release-latest-7.noarch.rpm 
[root@study2 tools]# rpm -qa epel-release
</pre>

- 安装cobbler和相关的依赖软件。

<pre>
[root@study2 tools]# yum -y install httpd dhcp tftp bind bind-chroot caching-nameserver python-ctypes cobbler cobbler-web pykickstart xinetd
[root@study2 tools]# rpm -qa httpd dhcp tftp bind bind-chroot caching-nameserver python-ctypes cobbler cobbler-web pykickstart xinetd 
</pre>

- 安装完毕后，我们在apache的配置文件目录里能看到cobbler的配置文件。

<pre>
[root@study2 tools]# ls /etc/httpd/conf.d/
autoindex.conf  cobbler.conf  cobbler_web.conf  README  ssl.conf  userdir.conf  welcome.conf
</pre>

- 启动apache和cobbler

<pre>
[root@cobbler ~]# systemctl start httpd
[root@cobbler ~]# systemctl start cobblerd
[root@cobbler-node1 ~]# netstat -ntlpua|grep httpd
tcp6       0      0 :::80                   :::*                    LISTEN      2668/httpd          
tcp6       0      0 :::443                  :::*                    LISTEN      2668/httpd          
[root@cobbler-node1 ~]# ps -ef|grep cobbler
apache     2669   2668  0 13:14 ?        00:00:00 (wsgi:cobbler_w -DFOREGROUND
root       2697      1  0 13:14 ?        00:00:00 /usr/bin/python2 /usr/bin/cobblerd -F
root       2731   2353  0 13:17 pts/1    00:00:00 grep --color=auto cobbler
</pre>

- 运行cobbler check，根据显示的提示，进行下一步操作。（只有启动了cobbler才能运行cobbler check）

###需要操作的选项有：
1. cobbler配置文件settings 
2. tftp配置文件 
3. cobbler get-loaders
4. start rsyncd.service
5. 设置系统的默认root密码

<pre>
[root@study2 tools]#  cobbler check          
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : SELinux is enabled. Please review the following wiki page for details on ensuring cobbler works correctly in your SELinux environment:
    https://github.com/cobbler/cobbler/wiki/Selinux
4 : change 'disable' to 'no' in /etc/xinetd.d/tftp
5 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
6 : enable and start rsyncd.service with systemctl
7 : debmirror package is not installed, it will be required to manage debian deployments and repositories
8 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
9 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
</pre>

## 配置文件修改
> 每次对cobbler相关配置文件的修改，都需要cobbler sync，才能使设置的参数生效。


- 修改cobbler的配置文件settings 272行和384行，修改tftp和server地址。

<pre>
[root@cobbler ~]# vim /etc/cobbler/settings
....
272 next_server: 10.0.0.201
...
384 server: 10.0.0.201
....
</pre>

- 修改tftp文件，设置为允许

<pre>
[root@cobbler-node1 ~]# sed -i 's@disable.*=.*@disable = no@g' /etc/xinetd.d/tftp 
[root@cobbler-node1 ~]# grep disable /etc/xinetd.d/tftp 
        disable = no
[root@cobbler-node1 ~]# systemctl start xinetd
[root@cobbler-node1 ~]# lsof -i udp:69
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
xinetd  11663 root    5u  IPv4  38158      0t0  UDP *:tftp 
</pre>


- 设置root密码

生成密码
<pre>
[root@cobbler-node1 ~]# openssl passwd -1 -salt 'cobbler' 'cobbler'     
$1$cobbler$M6SE55xZodWc9.vAKLJs6.
</pre>

修改cobbler的配置文件settings第101行,设置自动部署安装系统的root密码
<pre>
[root@cobbler-node1 ~]# vim /etc/cobbler/settings 
...
101 default_password_crypted: "$1$cobbler$M6SE55xZodWc9.vAKLJs6."
...
</pre>

- 让cobbler来管理dhcp

修改cobbler的配置文件settings第242行
<pre>
[root@cobbler ~]# vim /etc/cobbler/settings 
...
242 manage_dhcp: 1
...
</pre> 

修改cobbler的dhcp模版文件
<pre>
[root@cobbler ~]# vim /etc/cobbler/dhcp.template 
按需求进行修改。列如 网关、分配的ip范围，dns。
注意：所分配的地址必须和服务器在同一个网段内

 21 subnet 10.0.0.0 netmask 255.255.255.0 {
 22      option routers             10.0.0.1;
 23      option domain-name-servers 223.5.5.5;
 25      range dynamic-bootp        10.0.0.220 10.0.0.250;
</pre>



- 启动rsync

<pre>
[root@cobbler-node1 ~]# systemctl start rsyncd
[root@cobbler-node1 ~]# lsof -i :873
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
rsync   11654 root    4u  IPv4  36817      0t0  TCP *:rsync (LISTEN)
rsync   11654 root    5u  IPv6  36818      0t0  TCP *:rsync (LISTEN)
</pre>

- 运行cobbler get-loaders     

>如果显示Could not resolve host,需检查dns。

<pre>
[root@cobbler-node1 ~]# cobbler get-loaders                            
task started: 2016-10-12_121448_get_loaders
task started (id=Download Bootloader Content, time=Wed Oct 12 12:14:48 2016)
path /var/lib/cobbler/loaders/README already exists, not overwriting existing content, use --force if you wish to update
path /var/lib/cobbler/loaders/COPYING.elilo already exists, not overwriting existing content, use --force if you wish to update
downloading http://cobbler.github.io/loaders/COPYING.yaboot to /var/lib/cobbler/loaders/COPYING.yaboot
downloading http://cobbler.github.io/loaders/COPYING.syslinux to /var/lib/cobbler/loaders/COPYING.syslinux
downloading http://cobbler.github.io/loaders/elilo-3.8-ia64.efi to /var/lib/cobbler/loaders/elilo-ia64.efi
downloading http://cobbler.github.io/loaders/yaboot-1.3.17 to /var/lib/cobbler/loaders/yaboot
downloading http://cobbler.github.io/loaders/pxelinux.0-3.86 to /var/lib/cobbler/loaders/pxelinux.0
downloading http://cobbler.github.io/loaders/menu.c32-3.86 to /var/lib/cobbler/loaders/menu.c32
downloading http://cobbler.github.io/loaders/grub-0.97-x86.efi to /var/lib/cobbler/loaders/grub-x86.efi
downloading http://cobbler.github.io/loaders/grub-0.97-x86_64.efi to /var/lib/cobbler/loaders/grub-x86_64.efi
*** TASK COMPLETE ***
</pre>

- 重启 cobbler 服务并执行cobbler sync 让所有的修改生效。

<pre>
[root@cobbler-node1 ~]# systemctl restart cobblerd
[root@cobbler-node1 ~]# cobbler sync
task started: 2016-10-12_124651_sync
task started (id=Sync, time=Wed Oct 12 12:46:51 2016)
running pre-sync triggers
cleaning trees
removing: /var/lib/tftpboot/pxelinux.cfg/default
removing: /var/lib/tftpboot/grub/images
removing: /var/lib/tftpboot/grub/grub-x86.efi
removing: /var/lib/tftpboot/grub/grub-x86_64.efi
removing: /var/lib/tftpboot/grub/efidefault
removing: /var/lib/tftpboot/s390x/profile_list
copying bootloaders
copying: /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
copying: /var/lib/cobbler/loaders/menu.c32 -> /var/lib/tftpboot/menu.c32
copying: /var/lib/cobbler/loaders/yaboot -> /var/lib/tftpboot/yaboot
copying: /usr/share/syslinux/memdisk -> /var/lib/tftpboot/memdisk
copying: /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
copying: /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
copying distros to tftpboot
copying images
generating PXE configuration files
generating PXE menu structure
rendering TFTPD files
generating /etc/xinetd.d/tftp
cleaning link caches
running post-sync triggers
running python triggers from /var/lib/cobbler/triggers/sync/post/*
running python trigger cobbler.modules.sync_post_restart_services
running shell triggers from /var/lib/cobbler/triggers/sync/post/*
running python triggers from /var/lib/cobbler/triggers/change/*
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***


[root@study2 tools]# netstat -ntlpua|egrep "httpd|dhcpd|rsync|xinetd"
tcp        0      0 0.0.0.0:873             0.0.0.0:*               LISTEN      10284/rsync         
tcp6       0      0 :::80                   :::*                    LISTEN      10159/httpd         
tcp6       0      0 :::443                  :::*                    LISTEN      10159/httpd         
tcp6       0      0 :::873                  :::*                    LISTEN      10284/rsync         
udp        0      0 0.0.0.0:54265           0.0.0.0:*                           10339/dhcpd         
udp        0      0 0.0.0.0:67              0.0.0.0:*                           10339/dhcpd         
udp        0      0 0.0.0.0:69              0.0.0.0:*                           10245/xinetd        
udp6       0      0 :::3022                 :::*                                10339/dhcpd 
</pre>



## 导入系统镜像


- 导入centos 7的镜像
>插入7的光盘或镜像
<pre>
[root@cobbler-node1 ~]# mount /dev/cdrom /media/
[root@cobbler-node1 ~]# cobbler import --path=/media/ --name=Centos7_x86_64  --arch=x86_64    
task started: 2016-10-12_125851_import
task started (id=Media import, time=Wed Oct 12 12:58:51 2016)
Found a candidate signature: breed=redhat, version=rhel6
Found a candidate signature: breed=redhat, version=rhel7
Found a matching signature: breed=redhat, version=rhel7
Adding distros from path /var/www/cobbler/ks_mirror/Centos7_x86_64-x86_64:
creating new distro: Centos7-x86_64
creating new profile: Centos7-x86_64
associating repos
checking for rsync repo(s)
checking for rhn repo(s)
checking for yum repo(s)
starting descent into /var/www/cobbler/ks_mirror/Centos7_x86_64-x86_64 for Centos7-x86_64
processing repo at : /var/www/cobbler/ks_mirror/Centos7_x86_64-x86_64
need to process repo/comps: /var/www/cobbler/ks_mirror/Centos7_x86_64-x86_64
looking for /var/www/cobbler/ks_mirror/Centos7_x86_64-x86_64/repodata/*comps*.xml
Keeping repodata as-is :/var/www/cobbler/ks_mirror/Centos7_x86_64-x86_64/repodata
*** TASK COMPLETE ***
</pre>
    
- 导入centos 6的镜像
>插入6的光盘或镜像
<pre>
[root@cobbler-node1 ~]# umount /media/
[root@cobbler-node1 ~]# mount /dev/cdrom /media/
[root@cobbler-node1 ~]# cobbler import --path=/media/ --name=Centos6.6_x86_64  --arch=x86_64   
task started: 2016-10-12_130303_import
task started (id=Media import, time=Wed Oct 12 13:03:03 2016)
Found a candidate signature: breed=redhat, version=rhel6
Found a matching signature: breed=redhat, version=rhel6
Adding distros from path /var/www/cobbler/ks_mirror/Centos6.6_x86_64-x86_64:
creating new distro: Centos6.6-x86_64
trying symlink: /var/www/cobbler/ks_mirror/Centos6.6_x86_64-x86_64 -> /var/www/cobbler/links/Centos6.6-x86_64
creating new profile: Centos6.6-x86_64
associating repos
checking for rsync repo(s)
checking for rhn repo(s)
checking for yum repo(s)
starting descent into /var/www/cobbler/ks_mirror/Centos6.6_x86_64-x86_64 for Centos6.6-x86_64
processing repo at : /var/www/cobbler/ks_mirror/Centos6.6_x86_64-x86_64
need to process repo/comps: /var/www/cobbler/ks_mirror/Centos6.6_x86_64-x86_64
looking for /var/www/cobbler/ks_mirror/Centos6.6_x86_64-x86_64/repodata/*comps*.xml
Keeping repodata as-is :/var/www/cobbler/ks_mirror/Centos6.6_x86_64-x86_64/repodata
*** TASK COMPLETE ***
</pre>

- 查看导入的系统文件
<pre>
[root@cobbler-node1 ~]# ls /var/www/cobbler/ks_mirror
Centos6.6_x86_64-x86_64  Centos7_x86_64-x86_64  config
</pre>

## 其他高级操作
- cobbler常用命令
<pre>	
[root@cobbler-node1 ~]# cobbler profile
usage
=====
cobbler profile add
cobbler profile copy
cobbler profile dumpvars
cobbler profile edit
cobbler profile find
cobbler profile getks
cobbler profile list
cobbler profile remove
cobbler profile rename
cobbler profile report
</pre>

- 查看cobbler 已经导入的系统

<pre>
[root@cobbler-node1 ~]# cobbler profile list
   Centos6.6-x86_64
   Centos7-x86_64
</pre>

- 查看ks文件内容

<pre>
[root@cobbler-node1 ~]# cobbler profile report
Name                           : Centos7-x86_64
TFTP Boot Files                : {}
Comment                        : 
DHCP Tag                       : default
Distribution                   : Centos7-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/sample_end.ks
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : []
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 : 
Internal proxy                 : 
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      : 
Virt RAM (MB)                  : 512
Virt Type                      : kvm

Name                           : Centos6.6-x86_64
TFTP Boot Files                : {}
Comment                        : 
DHCP Tag                       : default
Distribution                   : Centos6.6-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/sample_end.ks
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : []
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 : 
Internal proxy                 : 
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      : 
Virt RAM (MB)                  : 512
Virt Type                      : kvm
</pre>

- 指定一个自定义的ks.cfg文件
> 请参照规范将文件放到/var/lib/cobbler/kickstarts目录下,然后执行如下操作。

<pre>
cobbler profile edit --name=Centos7-x86_64 --kickstart=/var/lib/cobbler/kickstarts/centos6.6.cfg
cobbler sync
</pre>


- 修改centos7的网卡名为eth开头

<pre>
[root@cobbler-node1 ~]# cobbler profile edit --name=Centos7-x86_64 --kopts='net.ifnames=0 biosdevname=0'
[root@cobbler-node1 ~]# cobbler profile report --name=Centos7-x86_64
Name                           : Centos7-x86_64
TFTP Boot Files                : {}
Comment                        : 
DHCP Tag                       : default
Distribution                   : Centos7-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {'biosdevname': '0', 'net.ifnames': '0'}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/sample_end.ks
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : []
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 : 
Internal proxy                 : 
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      : 
Virt RAM (MB)                  : 512
Virt Type                      : kvm

sync让配置生效
[root@cobbler ~]# cobbler sync
</pre>

**现在可以自动装机了, 开机F12**

- 自动重装系统
>注意此步在客户机上进行操作，客户端如果没能下载请使用阿里的yum源。
<pre>
[root@localhost ~]# yum -y install koan

查看cobbler能重装的系统
[root@localhost ~]# koan --server=10.0.0.201 --list=profiles
- looking for Cobbler at http://10.0.0.201:80/cobbler_api
Centos7-x86_64
Centos6.6-x86_64

 指定要重装的系统
[root@localhost ~]# koan --replace-self --server=10.0.0.201 --profile=Centos7-x86_64 

[root@localhost ~]# cat /etc/grub.conf 
# grub.conf generated by anaconda
#
# Note that you do not have to rerun grub after making changes to this file
# NOTICE:  You have a /boot partition.  This means that
#          all kernel and initrd paths are relative to /boot/, eg.
#          root (hd0,0)
#          kernel /vmlinuz-version ro root=/dev/mapper/VolGroup-lv_root
#          initrd /initrd-[generic-]version.img
#boot=/dev/sda
default=0
timeout=5
splashimage=(hd0,0)/grub/splash.xpm.gz
hiddenmenu
title kick1476198549
        root (hd0,0)
        kernel /vmlinuz_koan ro rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD rd_LVM_LV=VolGroup/lv_swap SYSFONT=latarcyrheb-sun16 crashkernel=auto rd_LVM_LV=VolGroup/lv_root  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet ksdevice=link lang= text net.ifnames=0 ks=http://10.0.0.201/cblr/svc/op/ks/profile/Centos7-x86_64 biosdevname=0 kssendmac
        initrd /initrd.img_koan
title CentOS 6 (2.6.32-504.el6.x86_64)
        root (hd0,0)
        kernel /vmlinuz-2.6.32-504.el6.x86_64 ro root=/dev/mapper/VolGroup-lv_root rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD rd_LVM_LV=VolGroup/lv_swap SYSFONT=latarcyrheb-sun16 crashkernel=auto rd_LVM_LV=VolGroup/lv_root  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet
        initrd /initramfs-2.6.32-504.el6.x86_64.img
</pre>

**重启后，就会自动重装系统**


- 修改cobbler网页密码（用于网页登录的用户名密码）

用户名/密码是cobbler，url是https://10.0.0.201/cobbler_web

修改密码为123
<pre>
[root@cobbler cobbler]# htdigest /etc/cobbler/users.digest "cobbler" cobbler 
Adding user cobbler in realm cobbler
New password: 
Re-type new password:
[root@cobbler cobbler]# cobbler sync
</pre>

- 修改grub选单（可选）
<pre>
 [root@cobbler cobbler]#  cd /etc/cobbler/pxe/
[root@cobbler pxe]# ll pxedefault.template 
-rw-r--r--. 1 root root 238 Jan 23  2016 pxedefault.template

我修改为如下：
+++
PROMPT 0
MENU TITLE damaicha | http://www.damaicha.org
TIMEOUT 200
TOTALTIMEOUT 6000
ONTIMEOUT $pxe_timeout_profile
LABEL local
        MENU LABEL (local)
        MENU DEFAULT
        LOCALBOOT -1
$pxe_menu_items
MENU end
+++
[root@cobbler pxe]# cobbler sync
</pre>

- 自定义yum源

<pre>
1.添加repo
[root@cobbler ~]# cobbler repo add --name=openstack-mitaka --mirror=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ --arch=x86_64 --breed=yum

2.同步repo
[root@cobbler ~]# cobbler reposync

3.添加repo到对应的profile
[root@cobbler pxe]# cobbler profile edit --name=Centos7-x86_64  --repos="openstack-mitaka"

4.修改kickstart文件，添加如下。
[root@cobbler ~]# vi /var/lib/cobbler/kickstarts/sample_end.ks
%post
添加这个，将将把yum库添加进去
$yum_config_stanza
%end
</pre>

- cobbler自定义安装
>大前提是需要知道mac地址和规划好相关的配置信息，如网络信息,主机名等。

主机信息：
00:50:56:3E:6A:CF   
规划信息：
10.0.0.202   linux-node2.oldboyedu.com   223.5.5.5
<pre>
[root@cobbler-node1 ~]# cobbler system add --name=linux-node2.oldboyedu.com\
--mac=00:50:56:3E:6A:CF   --profile=Centos7-x86_64  --ip-address=10.0.0.202\
--subnet=255.255.255.0 --gateway=10.0.0.1 --interface=eth0 --static=1\
--hostname=linux-node2.oldboyedu.com --name-servers="223.5.5.5"\
--kickstart=/var/lib/cobbler/kickstarts/sample_end.ks

[root@cobbler-node1 ~]# cobbler system list
   linux-node2.oldboyedu.com
[root@cobbler-node1 ~]# cobbler sync
[root@cobbler-node1 ~]#  tail -12  /etc/dhcp/dhcpd.conf 
group {
    host generic1 {
        hardware ethernet 00:50:56:3E:6A:CF;
        fixed-address 10.0.0.202;
        option host-name "linux-node2.oldboyedu.com";
        option subnet-mask 255.255.255.0;
        option routers 10.0.0.1;
        filename "/pxelinux.0";
        next-server 10.0.0.201;
    }
}
</pre>

**此时客户机，从pxe网络启动，将会自动的安装指定的系统，不需要任何选择。**

## api 接口

- 调用 api 接口自动安装

<pre>
api控制文件/etc/httpd/conf.d/cobbler.conf 

[root@cobbler ~]# grep ProxyPass /etc/httpd/conf.d/cobbler.conf 
ProxyPass /cobbler_api http://localhost:25151/
ProxyPassReverse /cobbler_api http://localhost:25151/
</pre>

- 使用api的小例子
<pre>
#!/usr/bin/env python
import xmlrpclib
server = xmlrpclib.Server("http://10.0.0.201/cobbler_api")
print server.get_distros()
print server.get_profiles()
print server.get_systems()
print server.get_images()
print server.get_repos()
</pre>
