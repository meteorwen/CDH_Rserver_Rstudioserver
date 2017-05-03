# 系统环境准备
1.	开机启动图形界面与shell文本界面默认修改
centos6 是通过修改 /ect/inittab 文件将  id:3:initdefault
centos7 开启启动init 3（dos） 5（win）是通过软连接的方式实现的：
	
<pre><code>
	ln -sf /lib/systemd/system/runlevel3.target /etc/systemd/system/default.target

</pre></code>
2.	创建一个非root用户，今后用这个用户user登录并操作相关内容
<pre><code>
	adduser user1
	passwd 12345678
	chmod -v u+w /etc/sudoers		//只读的权限，需要先添加w权限
	chmod -v u-w /etc/sudoers			//权限收回
</pre></code>
3.	修改/etc/sudoers 文件内容在：
<pre><code>
	root    ALL=(ALL)       ALL
	master   ALL=(ALL)       ALL    /添加这行，赋值给你master用户sudo权利
</pre></code>
4.  克隆之后的操作系统需要重新分配物理地址：
<pre><code>
	1、	删除 /etc/sysconfig/network-scripts/ifcfg-eth0
	文件中删除两行：UUID、网卡物理地址mac
	2、	删除文件 /etc/udev/rules.d/70-persitent-net.rules
	修改主机名 vi /etc/sysconfig/network
	修改hosts  vi /etc/hosts 文件最后一行：ip地址		主机名
	reboot
</pre></code>
5.  yum设置
<pre><code>
	$ yum clean all
	$ yum makecache
	$ yum update
	$ yum list
</pre></code>

## 1、环境配置
1.	使用root用户登录主节点服务器，设置主机服务器名为dsg。
<pre><code>
	$ vi /etc/sysconfig/network
HOSTNAME=dsgcm
</pre></code>
2.	增加CDH集群中所有服务器IP地址和主机名对应关系。
<pre><code>
		$ vi /etc/hosts
		192.168.1.111  dsg  dsg
		192.168.1.112  cdh01  cdh01
		192.168.1.113  cdh02  cdh02
		192.168.1.114  cdh03  cdh03
		192.168.1.115  cdh04  cdh04
		192.168.1.116  cdh05  cdh05
		127.0.0.1     localhost
</pre></code>
3.	进行网络重启，使主机配置生效。
<pre><code>
	$ service network restart
</pre></code>
## 2、SSH免密钥安装
1.	使用root用户登录主节点服务器，生成公钥和私钥文件id_rsa.pub和id_rsa。
<pre><code>
	$ ssh-keygen -t rsa
</pre></code>
2.	根据提示信息，按三次回车键。
<pre><code>
Generating public/private rsa key pair.
Enter file in which to save the key (/service/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /service/.ssh/id_rsa.
Your public key has been saved in /service/.ssh/id_rsa.pub.
The key fingerprint is:
b4:3b:ec:26:2a:27:fd:42:68:92:e1:c1:6f:9d:33:c3 service@service
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|.       .        |
|.o     . .       |
|.oo.o . S        |
|o.oo.E . .       |
| o.o  + +        |
|  o + ....       |
|   +.+.o.        |
+-----------------+
</pre></code>
3.	将生成的公钥文件id_rsa.pub拷贝到所有从节点服务器中。
<pre><code>
	$ ssh-copy-id -i /root/.ssh/id_rsa.pub root@从节点IP地址
</pre></code>
4.	验证ssh是登录所有从节点服务器是否需要密码。
<pre><code>
	$ ssh 从节点IP地址
</pre></code>
5.	如果直接登录到从节点服务器，则设置成功。	
## 3、关闭防火墙
1.	使用root用户登录主节点服务器，关闭并禁用防火墙。
<pre><code>
	$ service iptables stop
	$ chkconfig iptables off
</pre></code>
2.	关闭SELinux。
<pre><code>
	$ echo "SELINUX=disabled" > /etc/sysconfig/selinux
</pre></code>
## 4、安装JDK配置环境变量
1.	使用root用户登录主节点服务器，查看服务器中是否已安装jdk软件。
<pre><code>
	$ rpm -qa |grep jdk
</pre></code>
2.	如果服务器中已安装jdk软件，请先对其进行卸载。
<pre><code>
	$ rpm -e --nodeps jdk名称
</pre></code>
3.	使用yum安装jdk-7u79-linux-x64.rpm。
<pre><code>
	$ yum install jdk-7u79-linux-x64.rpm
</pre></code>
4.	安装完成后，为root用户的配置文件配置JAVA_HOME和PATH参数。
<pre><code>
	$ vi /etc/profile
	export JAVA_HOME=/usr/java/jdk1.7.0_79
	export PATH=$JAVA_HOME/bin:$PATH
</pre></code>
5.	保存文件，并使环境变量生效。
<pre><code>
	$ :wq!
	$ source /root/.bash_profile
	$ . /etc/profile
</pre></code>
6.	请根据以上步骤对所有从节点服务器安装JDK软件。

## 5、安装NTP
1.	使用root用户登录主节点服务器，配置ntp服务器配置文件/etc/ntp.conf。
<pre><code>
	$ vi /etc/ntp.conf
</pre></code>
2.	在配置文件中设置与ntp服务器相同网段内的其他服务器都可以通过ntp服务器进行时间同步。
<pre><code>
	restrict 127.0.0.1
	restrict -6 ::1
	restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
	server 127.127.1.0
	fudge 127.127.1.0 stratum 10
</pre></code>
3.	在ntp服务器中重启ntpd服务，并设置开机自启动。
<pre><code>
	$ service ntpd restart
	$ chkconfig ntpd on
</pre></code>
4.	使用root用户登录从节点服务器，配置/etc/ntp.conf文件，增加主机服务器。
<pre><code>
	$ vi /etc/ntp.conf
	server 0.centos.pool.ntp.org
	server 1.centos.pool.ntp.org
	server 2.centos.pool.ntp.org
	server 192.168.1.111
</pre></code>
5.	重启动ntpd服务，手动同步ntp服务器时间。（是因为没有配置DNS,不能链接外网）
<pre><code>
	$ service ntpd restart
	$ ntpdate -u 192.168.1.91
</pre></code>
6.	系统自动完成时间同步大概需要5-10分钟，可以通过ntpstat命令查看是否已完成同步。
<pre><code>
	$ ntpstat
	synchronised to NTP server (192.168.1.45) at stratum 12 
	   time correct to within 42 ms
	   polling server every 1024 s
</pre></code>
## 6、安装配置mysql
1.	使用root用户登录主节点服务器，查看服务器中是否已安装mysql软件。
<pre><code>
	$ rpm -qa |grep -i mysql
</pre></code>
2.	如果服务器中已安装mysql软件，请先对其进行卸载。
<pre><code>
	$ rpm -e --nodeps mysql名称
</pre></code>
3.	对mysql软件包进行解压。
<pre><code>
	$ tar -zxvf MySQL-5.5.53-el6.x86_64.rpm.tar
</pre></code>
4.	使用yum安装MySQL，请按照devel，client，server的顺序安装。
<pre><code>
	$ yum install MySQL-devel-5.5.53-el6.x86_64.rpm
	$ yum install MySQL-client-5.5.53-el6.x86_64.rpm
	$ yum install MySQL-server-5.5.53-el6.x86_64.rpm
</pre></code>
5.	安装完成后，启动mysql数据库。
<pre><code>
	$ service mysql start
</pre></code>
6.  创建MySQL用户dsg
<pre><code>
	$ mysql -uroot -p'root_password'  //登录
	mysql > CREATE USER 'username'@'host' IDENTIFIED BY 'password'; 
	mysql > grant all privileges on *.* TO 'dsg'@'%' with grant option;
</pre></code>
1.	使用root用户MySQL数据库。
<pre><code>
	$ mysql -uroot -p
</pre></code>
2.	创建hive数据库，并授权。
<pre><code>
	mysql> create database hive default charset latin1;
	mysql> grant all on hive.* to 'dsg'@'%' identified by 'dsg' with grant option;
	mysql> flush privileges;
</pre></code>
3.	创建hue数据库，并授权。
<pre><code>
	mysql> create database hue default charset utf8;
	mysql> grant all on hue.* to 'dsg'@'%' identified by 'dsg' with grant option;
	mysql> flush privileges;
</pre></code>
4.	创建oozie数据库，并授权。
<pre><code>
	mysql> create database oozie default charset utf8;
	mysql> grant all on oozie.* to 'dsg'@'%' identified by 'dsg' with grant option;
	mysql> flush privileges;
</pre></code>
5.	创建Activity Monitor集群监控数据库，并授权。
<pre><code>	
	mysql> create database amon default charset utf8;
	mysql> grant all on amon.* to 'dsg'@'%' identified by 'dsg' with grant option;
	mysql> flush privileges;
</pre></code>
6.	创建Reports Manager数据库，并授权。
<pre><code>
	mysql> create database rman default charset utf8;
	mysql> grant all on rman.* to 'dsg'@'%' identified by 'dsg' with grant option;
	mysql> flush privileges;
</pre></code>
## 7、安装依赖的第三方包
<pre><code>
	yum install chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse fuse-libs redhat-lsb –y
</pre></code>

# 安装CM 

## 部署Cloudera Manager Server
1.	将Cloudera Manager安装包cloudera-manager-el6-cm5.8.0_x86_64.tar.gz上传到主节点服务器中，并解压到/opt。
<pre><code>
$ tar -zxvf cloudera-manager-el6-cm5.8.0_x86_64.tar.gz -C /opt
</pre></code>
2.	解压后将生成cloudera和cm-5.8.0两个目录。
3.	将MySQL数据库的JDBC驱动mysql-connector-java-5.*.jar上传服务器，并拷贝到/opt/cm-5.8.0/share/cmf/lib目录中。
<pre><code>
$ cp mysql-connector-java-5.*.jar /opt/cm-5.11.0/share/cmf/lib
</pre></code>
4.	创建cloudera-scm用户。
<pre><code>
$ useradd --system --home=/opt/cm-5.8.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
useradd --system --home=/opt/cm-5.11.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
</pre></code>
5.	建立Cloudera Manager数据库cdhdb，并初始化该数据库。
<pre><code>
$ cd /opt/cm-5.11.0/share/cmf/schema/
$ ./scm_prepare_database.sh mysql -hdsgcm(数据库地址) -udsg(数据库用户) -pdsg(数据库密码) --scm-host dsgcm(cmserver服务器) cdhdb(数据库名) scm scm
./scm_prepare_database.sh mysql -hlocalhost -udsg -pdsg --scm-host localhost cdhdb scm scm
</pre></code>
*“cmserver用户名”输入Cloudera Manager Server服务器名或IP地址。*
6.	在/var/lib/中创建cloudera-scm-server目录，并改变该目录的拥有者改为cloudera-scm。
<pre><code>
$ mkdir /var/lib/cloudera-scm-server
$ chown cloudera-scm:cloudera-scm /var/lib/cloudera-scm-server
</pre></code>
7.	配置Agent，将/opt/cm-5.*.*/etc/cloudera-scm-agent/config.ini文件中的server_host修改为Cloudera Manager Server主机名。
<pre><code>
$ vi /opt/cm-5.11.0/etc/cloudera-scm-agent/config.ini
server_host=dsgcm(cm server主机名)
</pre></code>
8.	将CHD5相关的Parcel包放到主节点的/opt/cloudera/parcel-repo/目录中。
Parcel包含以下文件：
* CDH-5.8.0-1.cdh5.8.0.p0.18-el6.parcel.sha1
* manifest.json
<pre><code>
$ mv CDH-5.8.0-1.cdh5.8.0.p0.18-el6.parcel.sha1 CDH-5.8.0-1.cdh5.8.0.p0.18-el6.parcel.sha
</pre></code>
## 部署Cloudera Manager Agent

1.	将Cloudera Manager安装包cloudera-manager-el6-cm5.8.0_x86_64.tar.gz上传到从节点中，并解压到/opt。
<pre><code>
$ tar -zxvf cloudera-manager-el6-cm5.8.0_x86_64.tar.gz -C /opt
</pre></code>
2.	将MySQL数据库的JDBC驱动mysql-connector-java-5.1.35.jar上传服务器，并拷贝到/opt/cm-5.8.0/share/cmf/lib目录中。
<pre><code>
$ cp mysql-connector-java-5.1.35.jar /opt/cm-5.11.0/share/cmf/lib
</pre></code>
3.	配置Agent，将/opt/cm-5.8.0/etc/cloudera-scm-agent/config.ini文件中的server_host修改为主节点主机名。
<pre><code>
$ vi /opt/cm-5.11.0/etc/cloudera-scm-agent/config.ini

server_host=dsgcm(主节点主机名)
</pre></code>
4.	创建cloudera-scm用户。
<pre><code>
$ useradd --system --home=/opt/cm-5.11.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
</pre></code>
5.	在/var/lib/和/opt/cm-5.8.0/run/目录中创建cloudera-scm-agent目录。
<pre><code>
$ mkdir /var/lib/cloudera-scm-agent
$ mkdir /opt/cm-5.11.0/run/cloudera-scm-agent
</pre></code>
6.	请根据上面的步骤对所有从节点进行操作。
