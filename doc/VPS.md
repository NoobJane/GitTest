# VPS & SS搭建

自己动手，应有尽有。

一般的VPS操作系统支持CentOS，Debian和Ubuntu。推荐使用 CentOS 6 操作系统。登录VPS时，要使用root账号权限。

* 使用linux命令`vi`编辑文件，按`i`进行插入编辑。编辑完成后，先按`Esc`，再按`:`，输入`wq`即可保存退出。

## 安装SS

#### CentOS 6
1. 用yum安装python setuptools和pip：`yum install python setuptools && easy_install pip`。若是pip安装失败，则百度一下linux如何安装pip。
2. 安装SS：`pip install shadowsocks`

#### Debian / Ubuntu
1. 安装pip：`apt get install python pip` 
2. 安装SS：`pip install shadowsocks`

#### 修改配置
修改SS的配置文件：`vi /etc/shadowsocks.json`
```
{
 	"server":"my_server_ip",  #填入服务器的IP地址
	"local_address": "127.0.0.1",
	"local_port":1080,
	"port_password": {
  	    "8301": "foobar1",    #端口号，密码，可以多写几个
  	    "8302": "foobar2",
   	    "8303": "foobar3",
    	"8304": "foobar4"
	},
 	"timeout":300,
	"method":"aes-256-cfb",
 	"fast_open": false
}
```

#### 启动SS
启动分为两种。前端启动的话一旦出现什么问题断开，SS服务就会停止。如果使用的是后端启动的话，则不会轻易被断开。一般推荐使用后端启动的方法。

- 前端启动SS：`ssserver -c /etc/shadowsocks.json`
- 后端启动SS：`ssserver -c /etc/shadowsocks.json -d start`
- 后端停止SS：`ssserver -c /etc/shadowsocks.json -d stop`
- 后端重启SS：`ssserver -c /etc/shadowsocks.json -d restart`

<br>

## 配置防火墙
启动SS后，客户端能够通过ICMP连到VPS，但是不能通过TCP服务连接。而网站都是需要通过TCP进行连接的。此时应该查看防火墙配置，看是否已经开放了SS的端口，并使用的是否是TCP协议。

#### 修改防火墙配置文件

编辑查看防火墙设置：`vi /etc/sysconfig/iptables`。

该配置文件内容如下。
```
# Generated by iptables-save v1.4.7 on Mon Apr 15 09:40:49 2019
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT 
-A INPUT -p icmp -j ACCEPT 
-A INPUT -i lo -j ACCEPT 
# xshell连接使用
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT   
# 新添的SS端口，要开放多少个端口就该多少，改 "--dport" 部分
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8301 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8302 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited 
-A FORWARD -j REJECT --reject-with icmp-host-prohibited 
COMMIT
# Completed on Mon Apr 15 09:40:49 2019
```
保存退出后，重启防火墙，令配置生效（需要root权限）。

- 关闭防火墙：`service iptables stop`
- 开启防火墙：`service iptables start`
- 重启防火墙：`service iptables restart`


#### 配置文件参数说明
参考链接 —— [https://www.cnblogs.com/itxiongwei/p/5871075.html](https://www.cnblogs.com/itxiongwei/p/5871075.html)
- -A ：添加一条参数
- -p ：指定协议（比如TCP，ICMP）
- --dport ： 外部数据进入服务器为目标的端口
- --sport ： 服务器要发送数据到外部的端口
- -j ： 指定ACCEPT（接收），DROP（不接收）

<br>

## 优化：使用BBR加速
BBR加速可做可不做。参考链接 —— [https://www.cnblogs.com/Eason1024/p/8177665.html](https://www.cnblogs.com/Eason1024/p/8177665.html)

BBR是谷歌公司提出的一个开源TCP拥塞控制的算法。在最新的linux 4.9及以上的内核版本中已被采用。对于该算法的分析，ss不经过其它的任何的优化就能轻松的跑满带宽。（speedtest测试或fast测试）。

由于Google BBR非常新，任何低于4.9的linux内核版本都需要升级到4.9及以上才能使用，故若VPS本身内核版本较低的话，只有KVM架构的VPS才能使用本文升级内核并使用，openvz的VPS用户若内核版本较低则无法使用。

本文的BBR仅适用于 ++CentOS 6++ 操作系统。

1. 使用root用户登录。下载一键bbr脚本： `wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh`
2. 下载完成后，修改bbr脚本权限：`chmod +x bbr.sh`
3. 执行该bbr脚本：`./bbr.sh`
4. 执行完成后，提示重启VPS，输入`y`回车后重启。
5. 使用命令 `uname --r` 查看内核是否是4.9以上，返回4.9以上即可。
6. `sysctl net.ipv4.tcp_available_congestion_control`，返回含有`bbr cubic reno`表示成功。
7. 查看ipv4的tcp是否开启bbr：`sysctl net.ipv4.tcp_congestion_control `，返回含有`bbr`表示成功。
8. `sysctl net.core.default.qdisc`，返回`fq`成功。
9. 查看tcp模块是否启动bbr加速：`lsmod | grep bbr`，返回`tcp_bbr`表明启动成功。
