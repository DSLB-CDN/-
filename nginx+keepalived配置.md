# 一、换源(阿里云)

Centos 6
```
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
```
Centos 7
```
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```
生成缓存
```
yum makecache
```
参考链接:
[Centos的yum源更换为国内的阿里云源](https://www.cnblogs.com/zzsdream/p/7405083.html)
***

# 二、安装nginx
1、下载nginx源码<http://nginx.org/en/download.html>  
  
2、解压
```
tar -zxvf nginx-1.18.0.tar.gz
```
3、进入目录
```
cd nginx-1.18.0
```
4、由于四层负载均衡要使用stream模块，而默认情况下nginx没有编译进去，因此我们要手动添加。
```
./config --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-http_stub_status_module --with-stream
```
5、编译&&安装（如果提示缺少依赖，则通过`yum install`安装依赖后再进行编译安装）
```
make 

make install
```
6、默认情况下nginx会安装在 `/usr/local/nginx` 目录下，接下来修改配置文件。
```
cd /usr/local/nginx/conf

cp nginx.conf nginx.conf-bck        #备份下配置文件

vi nginx.conf
```
7、修改内容（参考）
```
worker_processes  1;                    #处理进程数

events {
    worker_connections  1024;           #最大连接数
}

stream{                                 #stream块
        upstream stream_backend{        
                hash $remote_addr consistent;
                #根据自己实际的上游服务器进行配置
                server 10.2.1.228:80;  
                server 10.2.1.229:80;
                server 10.2.1.230:80;
        }
        server{
                listen 80;
                proxy_connect_timeout 3s;
                proxy_timeout 3s;
                proxy_pass stream_backend;  
        }
}
```
8、启动nginx
```
cd /usr/local/nginx/sbin

./nginx
```
附下nginx常用指令
```
./nginx                 #启动nginx     
./nginx -s reload       #重启nginx
./nginx -s stop         #关闭nginx
ps -ef | grep nginx     #查看nginx进程
```
9、如果防火墙是开启的话，我们还要对外开放指定端口，其他机器才能访问到，在这里我们需要的开放的是80端口。当然，也可以直接关闭防火墙。。。
```
firewall-cmd --add-port=80/tcp --permannet  #开放80端口
firewall-cmd --reload                       #重启防火墙
```
常用防火墙指令
```
firewall-cmd --list-all                     #查看防火墙信息
firewall-cmd --state                        #查看防火墙状态

systemctl start firewalld                   #启动防火墙
systemctl stop firewalld                    #关闭防火墙
systemctl enable firewalld                  #开机启动防火墙
systemctl disable firewalld                 #开机关闭防火墙
```
10、参考链接：
nginx相关
<https://blog.csdn.net/qq_37345604/article/details/90034424>
<https://baijiahao.baidu.com/s?id=1621555946739445068&wfr=spider&for=pc>

防火墙相关
<https://www.cnblogs.com/liea/p/11871262.html>
***
# 三、keepalived
1、下载源码包:<https://www.keepalived.org/download.html>

2、解压
```
tar -zxvf keepalived-2.0.20.tar.gz
```
3、进入目录
```
cd keepalived-2.0.20
```
4、配置
```
./config --prefix=/usr/local/keepalived
```
5、编译&&安装
```
make && make install
```
6、在`etc`目录下创建文件夹
```
mkdir /etc/keepalived
```
7、拷贝配置文件
```
cp /usr/local/keepaliced/etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf
```
8、修改配置文件
```
vi /etc/keepalived/keepalived.conf
```
9、主节点参考配置文件
```
! Configuration File for keepalived

global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
   router_id NodeA                      #节点id
}

vrrp_script chk_ngx {                   #检测脚本
    script "/etc/keepalived/tesh.sh"    #脚本路径
    interval 3                          #执行间隔
    weight 2
}

vrrp_instance VI_1 {
    state MASTER            #主节点：MASTER  备份节点：BACKUP    
    interface ens160        #网络接口，通过ip addr指令查看
    virtual_router_id 51    #VRRP组名，同组必须一致
    priority 100            #优先级（1-254之间），主节点优先级必须大于备份节点优先级，254为最大优先级
    advert_int 1            #组播信息发送间隔，组内节点需一致
    authentication {        #组内节点必须一致
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.2.1.198          #指定的虚拟ip，组内节点必须一致
    }

    track_script {          #指定检测脚本
        chk_ngx    
    }
}
```
10、备份节点配置文件与主节点相似，仅存在几处修改
```
! Configuration File for keepalived

global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
   router_id NodeB                      #修改1
}

vrrp_script chk_ngx {                   
    script "/etc/keepalived/check.sh"    
    interval 3                          
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP            #修改2    
    interface ens160        #网络接口，视情况修改
    virtual_router_id 51    
    priority 99             #修改处3
    advert_int 1            
    authentication {       
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.2.1.198          
    }

    track_script {          
        chk_ngx    
    }
}
```
11、编辑检测脚本
```
vi /etc/keepalived/check.sh
```
参考检测脚本
```
 #!/bin/bash
    run=$(ps -C nginx --no-header | wc -l)
    if [ $[run] == $[0] ];then              #首先检测nginx进程是否存在
        service keepalived stop
    else
        A=$(curl 127.0.0.1/test/)           
        if [ "${A}" != "Hello" ];then       #然后检测nginx是否有响应
            pkill -9 nginx
            service keepalived stop
        fi
    fi
exit 0
```
为了配合`keepalived`检测脚本进行健康检查，还要在nginx的配置文件中的`server块`中添加以下内容
```
 location = /test/ {
    return 200 "Hello";
}
```
12、开启keepalived
```
service keepalived start
```
keepalived 常用指令
```
service keepalived start    #开启
service keepalived stop     #关闭
service keepalived restart  #重启
```
参考链接：
<https://blog.csdn.net/nimasike/article/details/51867046>
<http://tech.it168.com/a2018/1022/5078/000005078536.shtml>
***

