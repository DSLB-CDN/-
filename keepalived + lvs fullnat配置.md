# 一、查看系统内核版本
首先要查看自己的系统是否为centos6，内核版本是否为2.6，若不是则需重新安装对应版本的系统（centos7系统在内核编译阶段会报错）。
```
#查看系统
[root@localhost software]# lsb_release -d
Description:    CentOS release 6.2 (Final)
#查看内核版本
[root@localhost software]# uname -r
2.6.32-220.el6.x86_64
```
***
# 二、上网下载软件包
```
kernel-2.6.32-220.23.1.el6.src.rpm
Linux-2.6.32-220.23.1.el6.x86_64.lvs.src.tar.gz
Linux-2.6.32-220.23.1.el6.x86_64.rs.src.tar.gz
Lvs-fullnat-synproxy.tar.gz
```
下载地址:<http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY>
***
# 三、内核编译
1、获取Kernel源码
```
rpm -ivh kernel-2.6.32-220.23.1.el6.src.rpm
```
2、
```
cd ~/rpmbuild/SPECS
rpmbuild -bp kernel.spec
```
若提示缺依赖则需要通过`yum`安装相关依赖
```
yum -y install rpm-build
yum -y install rng-tools

yum -y install gcc patchutils xmlto asciidoc
yum -y install elfutils-libelf-devel zlib-devel 
yum -y install binutils-devel newt-devel 
yum -y install python-devel hmaccalc
```
3、当界面执行到如下提示时
```###
### Now generating a PGP key pair to be used for signing modules.
###
### If this takes a long time, you might wish to run rngd in the background to
### keep the supply of entropy topped up.  It needs to be run as root, and
### should use a hardware random number generator if one is available, eg:
###
###     rngd -r /dev/hwrandom
###
### If one isn't available, the pseudo-random number generator can be used:
###
###     rngd -r /dev/urandom
###
+ gpg --homedir . --batch --gen-key /root/rpmbuild/SOURCES/genkey
gpg: WARNING: unsafe permissions on homedir `.'
gpg: keyring `./secring.gpg' created
gpg: keyring `./pubring.gpg' created
```
新开一个终端,然后执行以下两个指令中的一个
```
rngd -r /dev/hwrandom
rngd -r /dev/urandom
```
4、进入`Lvs-fullnat-synproxy.tar.gz`存放目录
```
tar -zxvf Lvs-fullnat-synproxy.tar.gz
```
5、
```
cd ~/rpmbuild/BUILD/kernel-2.6.32-220.23.1.el6/linux-2.6.32-220.23.1.el6.x86_64

cp ~/software/lvs-fullnat-synproxy/lvs-2.6.32-220.23.1.el6.patch .  

patch -p1<lvs-2.6.32-220.23.1.el6.patch
```
6、修改配置文件
```
vi .config
```
修改
`CONFIG_IP_VS_TAB_BITS=20`
7、编译安装
```
make -j16

make modules_install

make install
```
8、修改文件
`vi /boot/grub/grub.conf`
```
1、修改 default=0
2、在Kernel一行的最后添加 nohz=off
```
9、重启`reboot`

10、查看内核版本
```
[root@localhost ~]# uname -r
2.6.32
```
至此内核安装成功
***
# 四、安装keepalived
1、进入第三阶段解压的Lvs-fullnat-synproxy文件夹
```
cd Lvs-fullnat-synproxy
```
2、解压`lvs-tools.tar.gz`工具包
```
tar -zxvf lvs-tools.tar.gz
```
3、
```
cd tools/keepalived 
```
4、安装依赖
```
yum -y install popt-devel openssl-devel
```
5、修改内核路径
```
./configure --with-kernel-dir="/lib/modules/`uname -r`/build"
```
正常情况下`Use IPVS Framework`为`Yes`
```
Keepalived configuration
------------------------
Keepalived version       : 1.2.2
Compiler                 : gcc
Compiler flags           : -g -O2
Extra Lib                : -lpopt -lssl -lcrypto 
Use IPVS Framework       : Yes
IPVS sync daemon support : Yes
IPVS use libnl           : No
Use VRRP Framework       : Yes
Use Debug flags          : No
```
6、
```
make 
make install
```
7、复制文件
```
mkdir /etc/keepalived

cp -a bin/keepalived /sbin/
cp -a keepalived/etc/init.d/keepalived.init /etc/init.d/keepalived
cp -a keepalived/etc/keepalived/keepalived.conf /etc/keepalived
cp -a keepalived/etc/init.d/keepalived.sysconfig /etc/sysconfig/keepalived
```
8、查看keepalived状态
```
service keepalived status
```
9、常用keepalived指令
```
service keepalived start
service keepalived stop
service keepalived restart
```
***
# 五、安装ipvsadm
1、进入之前解压的tools文件夹
```
cd ipvsadm
make 
make install
```
2、
```
service irqbalance start
chkconfig --level 2345 irqbalance on
```
3、修改系统配置
```
vi /etc/sysctl.conf
```
添加以下内容
```
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.core.netdev_max_backlog = 500000
```
***
# 六、安装quagga
1、进入之前解压的tools文件夹
```
cd quagga
```
2、
```
./configure --disable-ripd --disable-ripngd --disable-bgpd --disable-watchquagga --disable-doc  --enable-user=root --enable-vty-group=root --enable-group=root --enable-zebra --localstatedir=/var/run/quagga
```
3、
```
make 
make install
```
4、修改系统配置
```
vi /etc/sysctl.conf
```
修改
```
net.ipv4.ip_forward = 1
```
***
# 七、一些设置
1、关闭防火墙
```
service iptables stop
```
2、关闭网卡lro和gro
```
ethtool -K eth0 gro off
ethtool -K eht0 lro off
#注意eht0网卡，每台机器都可能不同
```
***
# 八、主备模式配置
1、修改keepalived配置文件
```
vi /etc/keepalived/keepalived.conf
```
主机配置如下
```
! Configuration File for keepalived

global_defs {
   router_id node01
}

local_address_group laddr_g1 {
        10.2.1.158
}

virtual_server_group shanks1 {
        10.2.1.197 443
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 88
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.2.1.197
    }
}

virtual_server 10.2.1.197 443 {
    delay_loop 6
    lb_algo rr
    lb_kind FNAT
    protocol TCP
    syn_proxy
    laddr_group_name laddr_g1
    real_server 10.2.1.228 443 {
        weight 1
        TCP_CHECK {
                connect_timeout 5
        }
   }
   real_server 10.2.1.230 443 {
        weight 10
        TCP_CHECK {
                connect_timeout 10
        }
    }
   real_server 10.2.1.229 443 {
        weight 10
        TCP_CHECK {
                connect_time 10
        }
   }
}
```
备机配置如下
```
! Configuration File for keepalived

global_defs {
   router_id node02     #与主机有区别
}

local_address_group laddr_g1 {
        10.2.1.157      #与主机有区别
}

virtual_server_group shanks1 {
        10.2.1.197 443
}

vrrp_instance VI_1 {
    state BACKUP        #与主机有区别
    interface eth1      #这台机器的网卡为eth1
    virtual_router_id 88
    priority 90         #优先级要低于主机
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.2.1.197
    }
}

virtual_server 10.2.1.197 443 {
    delay_loop 6
    lb_algo rr
    lb_kind FNAT
    protocol TCP
    syn_proxy
    laddr_group_name laddr_g1
    real_server 10.2.1.228 443 {
        weight 1
        TCP_CHECK {
                connect_timeout 5
        }
   }
   real_server 10.2.1.230 443 {
        weight 10
        TCP_CHECK {
                connect_timeout 10
        }
    }
   real_server 10.2.1.229 443 {
        weight 10
        TCP_CHECK {
                connect_time 10
        }
   }
}
```
2、查看日志
```
[root@localhost keepalived]# tail -f /var/log/messages
Aug  3 07:20:11 localhost Keepalived_healthcheckers: Activating healtchecker for service [10.2.1.229]:443
Aug  3 07:20:12 localhost Keepalived_vrrp: VRRP_Instance(VI_1) Transition to MASTER STATE
Aug  3 07:20:12 localhost Keepalived_healthcheckers: TCP connection to [10.2.1.229]:443 failed !!!
Aug  3 07:20:12 localhost Keepalived_healthcheckers: Removing service [10.2.1.229]:443 from VS [10.2.1.197]:443
Aug  3 07:20:12 localhost Keepalived_healthcheckers: TCP connection to [10.2.1.228]:443 failed !!!
Aug  3 07:20:12 localhost Keepalived_healthcheckers: Removing service [10.2.1.228]:443 from VS [10.2.1.197]:443
Aug  3 07:20:13 localhost Keepalived_vrrp: VRRP_Instance(VI_1) Entering MASTER STATE
Aug  3 07:20:13 localhost Keepalived_vrrp: VRRP_Instance(VI_1) setting protocol VIPs.
Aug  3 07:20:13 localhost Keepalived_vrrp: VRRP_Instance(VI_1) Sending gratuitous ARPs on eth0 for 10.2.1.197
Aug  3 07:20:13 localhost Keepalived_healthcheckers: Netlink reflector reports IP 10.2.1.197 added
Aug  3 07:20:18 localhost Keepalived_vrrp: VRRP_Instance(VI_1) Sending gratuitous ARPs on eth0 for 10.2.1.197

```
3、查看ip
```
[root@localhost keepalived]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UNKNOWN qlen 1000
    link/ether 00:50:56:84:90:4c brd ff:ff:ff:ff:ff:ff
    inet 10.2.1.158/24 brd 10.2.1.255 scope global eth0
    inet 10.2.1.197/32 scope global eth0
    inet6 fe80::250:56ff:fe84:904c/64 scope link
       valid_lft forever preferred_lft forever

```
我们可以发现，虚拟ip已经成功绑定到了主机网卡上。

关闭主机的`keepalived`
```
service keepalived stop
```
查看备机的ip
```
[root@localhost keepalived]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:50:56:84:b8:1e brd ff:ff:ff:ff:ff:ff
    inet 10.2.1.157/24 brd 10.2.1.255 scope global eth1
    inet 10.2.1.197/32 scope global eth1
    inet6 fe80::250:56ff:fe84:b81e/64 scope link
       valid_lft forever preferred_lft forever
```
我们可以发现，虚拟ip飘到了备机网卡上。

4、查看`ipvsadm`信息
```
[root@localhost keepalived]# ipvsadm -l
IP Virtual Server version 1.2.1 (size=1048576)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.2.1.197:https rr synproxy
  -> 10.2.1.230:https             FullNat 10     0          0

```
5、一些坑
```
在最开始配置keepalived.conf文件时，virtual_router_id 用的是默认
参数51，然后发现当主机关闭keepalived后，虚拟ip无法飘到备机上。后
来通过指令 tail -f /var/log/messages 查看发现VRRP异常。

上网查了下发现是同一网段下virtual_router_id重复导致的。。。

原来，我之前在同一网段中配置过其他集群的keepalived，virtual_router_id
用的是默认参数51，所以就冲突了。在将这里的virtual_router_id参数设置为
88后问题得到解决。
```
***
##### 至此，keepalived + lvs fullnat 主备模式配置完成。

参考链接：
1、<http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY>

2、<https://blog.51cto.com/9346709/2308392>

3、<https://blog.51cto.com/clovemfong/1201791>