# socks5-proxy

RUN
---

```
$ go get -v -u github.com/40t/socks5-proxy
$ cp -rf $(go env GOPATH)/bin/socks5-proxy /usr/local/bin
$ socks5-proxy
```

USAGE
-----

```
$ socks5-proxy -p [port] -a [username:password]
$ socks5-proxy  //default port is 9999
```

DEMO
----

![image](https://github.com/40t/socks5-proxy/raw/master/README.jpg)


玩客云搭建socks5服务
----

搭建方法很简单，直接拿的github上的项目来用 https://github.com/40t/socks5-proxy 需要提前安装好go环境，具体怎么安装go环境之前小姐姐的一篇搭建傻妞的文章上面有，不同cpu架构选择用不同架构的go安装包即可。

搭建步骤很简单，步骤如下
```
git clone https://ghproxy.com/github.com/40t/socks5-proxy.git /root/socks5
cd /root/socks5
go get -v -u github.com/40t/socks5-proxy
cp -rf $(go env GOPATH)/bin/socks5-proxy /usr/local/bin
socks5-proxy
```
socks5-proxy这个命令直接就启动socks5服务端了，默认端口是9999。但是我不想用这个端口，所以按照项目说明socks5-proxy -p [port] -a [username:password] 添加了启动参数
```
socks5-proxy -p 13200
```
即使用13200端口启动服务，如需配置用户名和密码就添加-a参数

保持后台运行的话，各种方法都有，我习惯于用pm2
```
pm2 start "socks5-proxy -p 13200"
```


家里没有公网ip怎么办务
----

这种情况就跟我一样了，家里没有公网ip，那服务器怎么连家里的socks5代理呢。曲线救国，frp内网穿透。在github上面https://github.com/fatedier/frp/releases 下载frp。服务器上面用到frps和frps.ini，客户端用frpc和frpc.ini

frps搭建

下载好frp之后压缩包上传到服务器上解压，修改frps.ini里面的内容
```
[common]
bind_port = frp所用的端口
bind_addr = 0.0.0.0
token = 设置自己的token
dashboard_port = frp面板的端口
dashboard_user = 面板用户名
dashboard_pwd = 面板密码
subdomain_host = frp用的泛域名
vhost_http_port = http穿透的端口
```
还需让frps能够开机自启，所以新建一个frps.service
```
vi /lib/systemd/system/frps.service
```
内容如下
```
[Unit]
Description=fraps service
After=network.target network-online.target syslog.target
Wants=network.target network-online.target

[Service]
Type=simple

#启动服务的命令（此处写你的frps的实际安装目录）
#例如我服务器上的frps位于/root/frp/下面
#那么我得ExecStart=/root/frp/frps -c /root/frp/frps.ini
ExecStart=/your/path/frps -c /your/path/frps.ini

[Install]
WantedBy=multi-user.target
```
好了这样就可以使能开机自启了

先启动frps
```
systemctl start frps
```
使能开机自启
```
systemctl enable frps
```
下一步，在玩客云上面配置frpc

玩客云是armv7架构，frp下载的时候要下载对应的架构，同样也是上传玩客云里解压出来，不过这回需要修改的是frpc.ini

内容如下
```
[common]
server_addr = //这里填frps服务器的地址或者域名
server_port = //frps上配置的使用端口，注意是使用端口不是http和https穿透端口也不是面板端口
token = //这个填frps服务器对应的token
login_fail_exit = false
[socks5]
type = tcp
local_ip = 127.0.0.1
local_port = 本地端口，如我用的socks5是13200则这里填13200
remote_port = 远程服务器上的端口号，即你想在frps上开放的端口号，这里我填的14200
```
同样需要配置开机自启，参照上面建立frpc.service即可，即将frps改成frpc

服务器上面polipo配置代理
----

服务器防火墙记得开放对应端口，我配置的远程端口为14200，即防火墙开放14200（这里云服务商提供的防火墙可以不用开放，只需要开放服务器系统上的防火墙）。查了些相关资料发现socks5并不是所有服务都能使用的，有的服务还不支持使用带用户认证的socks5，所以我选择用polipo将socks5转变成http在服务器上面使用（疯狂套娃）。

虽然ubuntu好像可以直接apt install polipo

不过我没试过，反正我在Debian上自带的软件源没法直接安装polipo，干脆编译安装吧
```
git clone https://ghproxy.com/github.com/jech/polipo.git /root/polipo
cd /root/polipo
make all
su -c 'make install'
```
在这里执行make的时候可能会报错，自己看一下

如果提示make cc:命令未找到
```
make install gcc 
make install gcc-c++
```
如果make : makeinfo : 命令未找到
```
yum install texinfo或者apt install texinfo
```
解决完报错后重新执行
```
make all
su -c 'make install'
```
再建立一下polipo的配置文件
```
mkdir /opt/polipo
vi /opt/polipo/config
```
内容如下
```
logSyslog = true
logFile = /var/log/polipo.log
socksParentProxy = "socks5服务端ip:端口"       //因为我用了frp内网穿透，所以这里我填的127.0.0.1:14200
socksProxyType = socks5
proxyPort = polipo的服务端口，自己设定一个     //这里我使用14300端口
proxyAddress = "0.0.0.0"
```
同样记得服务器自身系统防火墙开放14300端口。对于polipo的后台运行，可以参考建立polipo.service，不过这里我就要用pm2，就是玩。
```
pm2 start “/root/polipo/polipo -c /opt/polipo/config”
```
这样就可以给服务器配置对应的http代理，也可以用curl检测下你的端口是否是对应的代理，我这里就是检测服务器的14300端口
```
curl --connect-timeout 2 -x 127.0.0.1:14300 cip.cc
```
正确无误的话，应该显示的是家里的ip

青龙等容器使用代理
----
青龙好像是可以通过自身的配置文件去使用代理，不过我还是用通用的方法，即docker run命令上添加-e环境变量

比如青龙的docker run命令
```
docker run -dit \
  -v /home/qinglong:/ql/data \
  -p 5700:5700 \
  -e http_proxy="http://172.17.0.1:14300" \
  -e https_proxy="http://172.17.0.1:14300" \
  --name qinglong \
  --hostname qinglong \
  --restart unless-stopped \
  whyour/qinglong:latest
```
这里的172.17.0.1其实指的是docker里面宿主机的ip也就是服务器系统本体。对于有的项目需要验证ip，用家宽代理ip总会变化可能会造成封掉授权，所以需要额外添加-e no_proxy参数，即-e no_proxy="验证中心域名"防止在验证授权的时候ip总在变化。

这个方法很套娃了，谁让我本地没啥好机器跑呢，手套就有玩客云，和香橙派zero，还都是老arm32位的机器，又没公网ip，难顶的很。
