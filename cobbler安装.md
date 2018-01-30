#cobbler安装
###前言
运维自动化对系统管理员十分重要性，尤其是对于在服务器数量按几百台、几千台增加的公司而言，单单是装系统，如果不通过自动化来完成，根本是不可想象的，下面我们介绍Cobbler。

###Cobbler介绍

Cobbler是一个快速网络安装linux的服务，而且在经过调整也可以支持网络安装windows。该工具使用python开发，小巧轻便（才15k行代码），使用简单的命令即可完成PXE网络安装环境的配置，同时还可以管理DHCP，DNS，以及yum包镜像。Cobbler支持命令行管理，web界面管理，还提供了API接口，可以方便二次开发使用。和Kickstart不同的是，使用cobbler不会因为在局域网中启动了dhcp而导致有些机器因为默认从pxe启动在重启服务器后加载tftp内容导致启动终止。

此次安装的系统为：

CentOS release 6.9 (Final)

安装服务，关闭selinux

禁用selinux：

Setenforce 0   临时禁用，重启失效，永久生效需要修改以下：

![](https://i.imgur.com/W8qjojK.png)

Shutdown -r now 重启系统

额外需要的服务还有tftp，rsync，xinetd，httpd。所以如果安装系统的时候如果这几个包没装上，请手动安装。

yum install tftp-server rsync xinetd httpd pykickstart dhcp

chkconfig xinetd on

chkconfig tftp on

service xinetd start

Cobber：协同各个模块共同完成操作系统部署的一个平台
 
httpd： 为cobbler提供一个可以使用http访问的界面 

rsync、tftp-server：用于在客户机启动时为客户机传输启动镜像及安装文件

xinetd：超级守护进程，用于管理rysnc和tftp这两个瞬时守护进程

dhcp：为要安装OS的机器启动时分配IP地址

python-cypes：python的一个外部库，提供和C语言兼容的数据类型

##cobbler安装

Yum 源安装

 rpm -ivh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

You could try using --skip-broken to work around the problem

You could try running: rpm -Va --nofiles --nodigest

解决方法如下：

yum clean all

rpm --rebuilddb

yum update

yum install cobbler -y

如果想要web界面还需要安装cobbler-web

yum install cobbler-web -y  此yum源没有cobbler-web的安装包

启动cobbler,启动httpd服务

/etc/init.d/cobblerd start

/etc/init.d/httpd start



检查配置，执行（如果check完有下列报错，请执行cobbler重启）

![](https://i.imgur.com/YzV18UZ.png)

cobbler check   （不同的系统check出来信息是不一样的，请仔细核对自己的信息，按照信息提示修改）
![](https://i.imgur.com/T3lX0Ku.png)
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.

2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.

3 : SELinux is enabled. Please review the following wiki page for details on ensuring cobbler works correctly in your SELinux environment:
https://github.com/cobbler/cobbler/wiki/Selinux

4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.

5 : change 'disable' to 'no' in /etc/xinetd.d/rsync

6 : since iptables may be running, ensure 69, 80/443, and 25151 are unblocked

7 : debmirror package is not installed, it will be required to manage debian deployments and repositories

8 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one

9 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.

根据check的内容，使用cobbler需要完成的9个步骤

修改 vim /etc/cobbler/settings

1) 找到server这行，将ip地址修改，server参数的值为提供cobbler服务的主机相应的IP地址或主机名（server:）
![](https://i.imgur.com/vRQFG9r.png)

2) 找到next_server这行，将ip地址修改，next_server参数的值为提供PXE服务的主机相应的IP地址 （next_server:）
![](https://i.imgur.com/qqVqesp.png)
3) 关闭并确认SELinux 处于关闭状态
Getenforce 查看selinux状态
![](https://i.imgur.com/e4TmpTT.png)

临时关闭setenforce 0

vi /etc/sysconfig/selinux

SELINUX=disabled #修改为disabled
![](https://i.imgur.com/JJCeSeI.png)

4)执行 cobbler get-loaders 命令 
![](https://i.imgur.com/G8I3OQH.png)

5) vim /etc/xinetd.d/rsync
将disable设置为no
![](https://i.imgur.com/9SGSCk4.png)

6) 放行防火墙端口 69，80/443，和25151
 vim /etc/sysconfig/iptables
![](https://i.imgur.com/Va8dYbb.png)

-A INPUT -m state --state NEW -m tcp -p tcp --dport 69 -j ACCEPT

-A INPUT -m state --state NEW -m udp -p udp --dport 69 -j ACCEPT

-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT

-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT

-A INPUT -m state --state NEW -m tcp -p tcp --dport 25151 -j ACCEPT

重启防火墙/etc/init.d/iptables restart

7)生成一串密码
 openssl passwd -1 -salt 'cobbler' 'cobbler'
![](https://i.imgur.com/4vn8a3l.png)
vim /etc/cobbler/settings

将生成的密码写入default_password_crypted
![](https://i.imgur.com/jaSXFrq.png)

8) yum -y install cman fence-agents
9) 重启/etc/init.d/cobblerd restart

配置dhcp
vim /etc/cobbler/settings 
将manage_dhcp:的值改成1
![](https://i.imgur.com/hyoBmhN.png)

修改dhcp的模板文件
vim /etc/cobbler/dhcp.template   （安装自己的需求修改）
![](https://i.imgur.com/F5ypM3J.png)
subnet 192.168.30.0 netmask 255.255.255.0 { #设置网段

option routers             192.168.30.1; #设置网关

option domain-name-servers 192.168.30.5,192.168.30.6; #设置dns服务器地址

option subnet-mask         255.255.255.0; #设置子网掩码

range dynamic-bootp        192.168.30.60 192.168.30.70;  #设置dhcp服务器IP地址租用的范围

default-lease-time         21600;  #默认租约时间

max-lease-time             43200;  #最大租约时间

next-server                $next_server;

重启cobbler

/etc/init.d/cobblerd restart

启动xinetd

/etc/init.d/xinetd start

同步cobbler

cobbler sync
![](https://i.imgur.com/Gzse7r2.png)

cat /etc/dhcp/dhcpd.conf

查看生成的dhcp配置文件
![](https://i.imgur.com/eI32sFq.png)

##管理cobbler

此挂载是挂载的本机系统的镜像，一个范例：

mount /dev/cdrom /mnt/  #挂在ISO光盘至服务器

cobbler import --path=/mnt/ --name=CentOS-7.1-x86_64 --arch=x86_64  # 导入镜像文件


--path 镜像路径   （/usr/local/src/） 

--name 为安装源定义一个名字

--arch 指定安装源是32位、64位、ia64, 目前支持的选项有: x86│x86_64│ia64


镜像存放目录，cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirror下的CentOS-7.1-x86_64-distro-x86_64目录下。因此/var/www/cobbler目录必须具有足够容纳安装文件的空间。

实例挂载：

挂载系统安装镜像到http服务器站点目录

上传系统安装镜像文件CentOS-6.5-x86_64-minimal.iso到/usr/local/src/目录

上传系统安装镜像文件CentOS-7-x86_64-Minimal-1708.iso到/usr/local/src/目录

mkdir -p /var/www/html/os/centos-6.5-x86_64  #创建挂载目录

mkdir -p /var/www/html/os/centos-7.0-x86_64  #创建挂载目录

 mount -t iso9660 -o loop /usr/local/src/CentOS-6.5-x86_64-minimal.iso /var/www/html/os/centos-6.5-x86_64/ #挂载系统镜像

mount -t iso9660 -o loop /usr/local/src/CentOS-7-x86_64-Minimal-1708.iso /var/www/html/os/centos-7.0-x86_64/ #挂载系统镜像

vi /etc/fstab   #添加以下代码。实现开机自动挂载
![](https://i.imgur.com/tfShstu.png)

/usr/local/src/CentOS-6.5-x86_64-minimal.iso /var/www/html/os/centos-6.5-x86_64/ iso9660  defaults,ro,loop  0 0

/usr/local/src/CentOS-7-x86_64-Minimal-1708.iso /var/www/html/os/centos-7.0-x86_64/ iso9660  defaults,ro,loop  0 0

备注：iso9660使用df  -T 查看设备  卸载：umount  /var/www/html/os/CentOS-5.10-x86_64

重复上面的操作，把自己需要安装的CentOS系统镜像文件都挂载到/var/www/html/os/目录下

cobbler import --path=/var/www/html/os/centos-6.5-x86_64 --name=centos-6.5-x86_64 --arch=x86_64 # 导入镜像文件
![](https://i.imgur.com/5HPtCgW.png)

 cobbler import --path=/var/www/html/os/centos-7.0-x86_64 --name=centos-7.0-x86_64 --arch=x86_64  # 导入镜像文件
![](https://i.imgur.com/BT5PBX9.png)

##管理profile

cobbler profile

![](https://i.imgur.com/s25COCB.png)

 cobbler profile list 查看导入的镜像文件

![](https://i.imgur.com/kfqXM0S.png)

 cobbler profile report  查看profile的内容

![](https://i.imgur.com/lidRGQr.png)
cobbler profile edit --name=centos-6.5-x86_64 --kickstart=/var/lib/cobbler/kickstarts/centos-6.5-x86_64

cobbler profile edit --name=centos-7.0-x86_64 --kickstart=/var/lib/cobbler/kickstarts/centos-7.0-x86_64

修改名称为CentOS-7.1-x86_64和CentOS-6.8-x86_64的自定义的kickstart文件

centos-6.5-x86_64文件

```

```

centos-7.0-x86_64文件

```

```
![](https://i.imgur.com/Zeik2Tl.png)
cobbler profile edit --name=centos-7.0-x86_64 --kopts='net.ifnames=0 biosdevname=0'

修改centos7内核
![](https://i.imgur.com/qk2h1rf.png)

cobbler profile report centos-7.0-x86_64 查看centos-7.0-x86_64的更改内容是否完成

cobbler sync   每次修改profile都需要同步

##cobbler部署操作系统

如果客户端连接出现TFTP OPen timeout  肯定是tftp的问题，可以先尝试关闭防火墙
