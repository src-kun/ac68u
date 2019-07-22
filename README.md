# ac68u
ac68u ss + chinadns + redsocks 全局ss代理

## 1  开启华硕原厂JFFS2，开启后重启路由后系统不会还原数据。
- 开启了SSH后。用SSH登录路由器。

```bash
nvram set jffs2_on=1
nvram set jffs2_enable=1
nvram set jffs2_format=1
nvram set jffs2_scripts=1
nvram commit
```
- 重启

```bash
reboot
```

- 验证JFFS2是否开启成功

```bash
nvram show | grep "jffs"
```
下面是开启的结果：
```
jffs2_on=1
jffs2_exec=
jffs2_enable=1
jffs2_size=65798144
jffs2_format=0
jffs2_scripts=1
log_path=/jffs
size: 55375 bytes (10161 left)
```
## 2  安装entware
启用了 JFFS 后，才可安装entware。 安装方法如下：
- 插入U盘，U盘格式化为 EXT2 、 EXT3 或 EXT4 都可以。格式化的方法，可以用电脑格式化。或在SSH下面。

```bash
mount #查看分区格式是否ext2或ext3 格式,如果不是,需要将将sda1格式化为ext2或ext3.
umount /mnt/sda1  #先卸载才能格式化 umount /tmp/mnt/awrt
mkfs.ext3 /dev/sda1 #格式化成ext3.
mount /dev/sda1 /mnt/sda1 #重新挂载好
```
- 在线安装，梅林系统内置命令

```bash
entware-setup.sh
```

- 离线安装

```
rm /tmp/opt
ln -s /mnt/sda1/entware /tmp/opt
TODO 下载lib直接复制到文件夹，如果联网不上可以试试先离线安装privoxy
```

## 3  安装shadowsocks、redsocks、chinadns

```bash
opkg install shadowsocks redsocks chinadns
```

## 4 启动配置服务
所有访问墙外的流量最后都会转到shadowsocks，`ac68u路由器`通过iptables将所有流量转到redsocks，redsocks再转发到shadowsocks完成翻墙，gfw会对禁止访问的域名进行dns污染可能找不到真实ip，使用chinadns是防止dns污染。访问链路如下：

`tcp ==>  redsocks ==> ss-local ==> 墙外`

`dns query ==> dnsmasq:53 ==> chinadns:1053 ==> ss-tunnel:5353 ==> 8.8.8.8`
### 4.1 修改配置，文件名、ip和端口按照实际情况修改
 - shadowsocks 
 
修改配置 /opt/etc/shadowsocks.json
```bash
{
 "server":"47.x.x.x",
 "server_port":5334,
 "local_address": "127.0.0.1",
 "local_port":1080,
 "password":"xxxx",
 "timeout":300,
 "method":"aes-256-cfb",
 "workers": 1
}

```

opkg安装ss后会自动加入开机启动中/opt/etc/init.d/S22shadowsocks无需改动，新建/opt/etc/init.d/S22sstunnel开机自启动ss-tunnel，ss-tunnel用于转发dns查询，开启代理后所有dns全部都走5353端口：
```bash
#!/bin/sh

ENABLED=yes
PROCS=ss-tunnel
ARGS="-s ss_server_ip -p 8334 -b 0.0.0.0 -l 5353 -k password -m aes-256-cfb -L 8.8.8.8:53 -u"
PREARGS=""
DESC=$PROCS
PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

[ -z "$(which $PROCS)" ] && exit 0

. /opt/etc/init.d/rc.func
```


- chinadns

修改/opt/etc/init.d/S56chinadns，指定dns查询的上级dns，国内会查询114.114.114.114，国外会查询127.0.0.1:5353==8.8.8.8:53，防止dns污染：
```bash
#!/bin/sh

ENABLED=yes
PROCS=chinadns
ARGS="-s 114.114.114.114,127.0.0.1:5353 -l /opt/etc/chinadns_iplist.txt -c /opt/etc/chinadns_chnroute.txt -p 1053"
PREARGS=""
DESC=$PROCS
PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

. /opt/etc/init.d/rc.fun
```
下载更新国内IP段：
```
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /opt/etc/chinadns_chnroute.txt
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /opt/etc/chinadns_iplist.txt
```

- redsocks

修改 /opt/etc/redsocks.conf，tcp流量全部走31338
```bash
base{
log_debug = off;
log_info = on;
log = "file:/mnt/sda1/log/redsocks.log";
daemon = on;
redirector = iptables;
user = nobody;
group = nobody;
}
redsocks {
local_ip = 0.0.0.0;
local_port = 31338;
ip = 127.0.0.1;
port = 1080;
type = socks5;
}

```
- dnsmasq

修改 /opt/etc/dnsmasq.conf，1053端口是chaindns的dns查询端口，dnsmasq:53 ==> chaindns:1053 ==> ss-tunnle:5353 ==> 8.8.8.8:53
```bash
conf-dir=/opt/etc/dnsmasq.d
no-resolv
server=127.0.0.1#1053
```

- ipset

使用ipset主要是为了访问国内ip时不走代理，提升访问国内网站的速度。

```bash
cd /opt/tmp/
curl -sL http://f.ip.cn/rt/chnroutes.txt | egrep -v '^$|^#' > cidr_cn
ipset -N cidr_cn hash:net
for i in `cat cidr_cn`; do echo ipset -A cidr_cn $i >> ipset.sh; done
rm -f ipset.sh cidr_cn
ipset save > /opt/etc/ipset.conf
```
开机启动/opt/etc/init.d/S20ipset，第一次设置完ipset保存规则`ipset save > /opt/etc/iptables.conf`，设置开机自动加载：
```
#!/bin/sh

ipset restore -f /opt/etc/ipset.conf
```

- iptables

ssup.sh
```bash
#!/bin/bash

SOCKS_SERVER=x.x.x.x # SOCKS 服务器的 IP 地址
# Setup the ipset
ipset restore -f /opt/etc/ipset.conf

# 在nat表中新增一个链，名叫：SHADOWSOCKS
iptables -t nat -N SHADOWSOCKS

# Allow connection to the server
iptables -t nat -A SHADOWSOCKS -d $SOCKS_SERVER -j RETURN

# Allow connection to reserved networks
iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN

# Allow connection to chinese IPs
iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set cidr_cn dst -j RETURN
# 如果你想对 icmp 协议也实现智能分流，可以加上下面这一条
iptables -t nat -A SHADOWSOCKS -p icmp -m set --match-set cidr_cn dst -j RETURN

# Redirect to Shadowsocks
# 把1081改成你的shadowsocks本地端口
iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-port 31338
# 如果你想对 icmp 协议也实现智能分流，可以加上下面这一条
iptables -t nat -A SHADOWSOCKS -p icmp -j REDIRECT --to-port 31338

# 将SHADOWSOCKS链中所有的规则追加到OUTPUT链中
iptables -t nat -A OUTPUT -p tcp -j SHADOWSOCKS
# 如果你想对 icmp 协议也实现智能分流，可以加上下面这一条
iptables -t nat -A OUTPUT -p icmp -j SHADOWSOCKS

# 内网流量流经 shadowsocks 规则链
iptables -t nat -A PREROUTING -s 192.168/16 -j SHADOWSOCKS
# 内网流量源NAT
iptables -t nat -A POSTROUTING -s 192.168/16 -j MASQUERADE
```

ssdown
```bash
#!/bin/bash

iptables -t nat -D OUTPUT -p icmp -j SHADOWSOCKS
iptables -t nat -D OUTPUT -p tcp -j SHADOWSOCKS
iptables -t nat -F SHADOWSOCKS
iptables -t nat -X SHADOWSOCKS
ipset destroy cidr_cn
```

iptablest 开机自动加载iptables的ss代理规则，第一次设置完iptables规则后保存到`iptables-save > /opt/etc/iptables.conf`，开机自动加载规则/opt/etc/init.d/S21iptables 
```bash
#!/bin/sh

iptables-restore /opt/etc/iptables.conf
```

## 更新
gfw相关IP段 https://github.com/felixonmars/dnsmasq-china-list

dnsmasq防污染配置文件：https://cokebar.github.io/gfwlist2dnsmasq/dnsmasq_gfwlist.conf

## 参考

https://w2x.me/2018/12/26/%E8%B7%AF%E7%94%B1%E5%99%A8%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86/

https://medium.com/@oliviaqrs/%E5%88%A9%E7%94%A8shadowsocks%E6%89%93%E9%80%A0%E5%B1%80%E5%9F%9F%E7%BD%91%E7%BF%BB%E5%A2%99%E9%80%8F%E6%98%8E%E7%BD%91%E5%85%B3-fb82ccb2f729

https://github.com/RMerl/asuswrt-merlin/wiki/Compile-Firmware-from-source-using-Ubuntu

https://github.com/gygy/asus_factory_image
