---
layout: post
title: 在路由器上创建全局代理模式
---

在路由器中做全局代理进行翻墙, 是一种很不错的方案, 这样可以让内网所有设备畅享互联网, 下面介绍一种我个人比较常用的全局代理方案

运行在路由器中的 [proxy_server](https://github.com/Jackarain/proxy) 配置文件内容:

``` bash
server_listen=0.0.0.0:1080
auth_users=
proxy_pass=https://jack:1111@example.remote.com:443
proxy_pass_ssl=true
http_doc=/tmp/
disable_udp=true
```

在上面配置中, auth_users= 特意使用无用户密码认证, 因为是面向内网的嘛, 也就用不着认证了.

server_listen 监听在 1080 端口, 这样也可以让内网用户直接连接到路由器上的 `proxy_server` 服务.

http_doc 实际上位于内网, 并不需要设置这个, 这里随便设置为 /tmp 目录.

以上, 路由器上的 `proxy_server` 服务配置完成了, 接下来运行它即可:

``` bash
proxy_server --config /etc/proxy-server/server.conf --disable_logs true
```

接下来配置运行在路由器上的 `tun2socks` 服务, 这里我推荐使用 [https://github.com/xjasonlyu/tun2socks](https://github.com/xjasonlyu/tun2socks)
它是一个经作者长期坚持不懈维护良好的软件.

`tun2socks` 运行参数如下(通常需要作为服务运行在路由器中):

``` bash
tun2socks -device tun3 -proxy socks5://127.0.0.1:1080 -tun-post-up "/bin/sh /etc/tun2socks/tun2socks.sh"
```

其中参数 -proxy socks5://127.0.0.1:1080 指定了连接 proxy_server 的 socks5 端口.

其中参数 -tun-post-up 是指定 tun2socks 运行后需要执行的脚本, tun2socks.sh 是一个可执行 shell 脚本, 内容如下:

``` bash
echo "start tun2socks..."

# 给 tun3 设备配置 ip 地址为 172.16.8.1 然后启用 tun3 设备
#
ip addr add 172.16.8.1/24 dev tun3
ip link set dev tun3 up

# 注意, example.remote.com 这里需要修改为 proxy_server 连接的上游代理服务的 ip
# 这样避免上游的 proxy_server 服务地址路由通过 172.16.8.1
# pppoe-WAN 是路由器外网网络接口
route add -host <example.remote.com> dev pppoe-WAN

# google / telegram 等服务的 ip 段都通过 tun2socks
#
route add -net 103.235.0.0/16 gw 172.16.8.1
route add -net 117.168.0.0/16 gw 172.16.8.1
route add -net 117.18.0.0/16 gw 172.16.8.1
route add -net 149.154.0.0/16 gw 172.16.8.1
route add -net 185.199.0.0/16 gw 172.16.8.1
route add -net 74.86.0.0/16 gw 172.16.8.1
route add -net 91.108.0.0/16 gw 172.16.8.1
route add -net 8.8.4.0/24 gw 172.16.8.1
route add -net 8.8.8.0/24 gw 172.16.8.1

# 自行添加其它各种国外地址段, 可通过 https://bgp.he.net/ 查询一些企业组织的 ip 段然后添加至此...

```

至上, 全局代理部分已经完成, 还剩下一个事情就是内网机器的 dns 查询问题, 因为像 google.com 之类的域名在国内通常是被污染的, dns 查询结果并不靠谱, 这里我推荐使用 `dnscrypt-proxy` 来创建一个 dns 服务用于替代运营商给定的 dns 服务.

dnscrypt-proxy 用法很简单, 运行参数通常为:

``` bash
dnscrypt-proxy -config /etc/dnscrypt-proxy/dnscrypt-proxy.toml
```

`dnscrypt-proxy.toml` 是 `dnscrypt-proxy` 的参数配置文件, 可以参考官方文档, 通常不需要修改什么, 无非就是 `server_names` 这个参数有时需要修改.

通常配置 `server_names = [ 'google', 'cloudflare', 'cloudflare-ipv6' ]` 就可以了,

另外 `listen_addresses` 需要设置, 我这里设置为

``` bash
listen_addresses = ['[::]:5003']
listen_addresses = ['0.0.0.0:5003']
```

如果直接使用 dnscrypt-proxy 来解析域名, 那么会因为 `dnscrypt-proxy` 导致国内网站的域名被解析到国外IP这种情况, 如果路由器上 dns 解析是使用 `dnsmasq` 的话, 则可以很简单的使用 [https://github.com/cokebar/gfwlist2dnsmasq](https://github.com/cokebar/gfwlist2dnsmasq) 这个项目来将 gfwlist 转换成 `dnsmasq` 的配置文件:

``` bash
./gfwlist2dnsmasq.sh -d 127.0.0.1  -p 5003 -o gfwlist.conf
```

将上面生成的 gfwlist.conf 文件放入 `/etc/dnsmasq.d` 或作为 `/etc/dnsmasq.servers` 配置给 `dnsmasq`, 它将自动将 gfwlist.conf 配置文件中的域名请求通过 127.0.0.1:5003 (即 dnscrypt-proxy) 来查询, 而配置文件外的域名则通过默认 dns 服务器查询.

除了使用 dnsmasq 之外, 还有一种域名解决方案就是通过 `chinadns-ng` 来分流 dns 查询请求, 下面是具体做法:

在 `dnscrypt-proxy` 之前再运行一个 `chinadns-ng` 用于将绝大部分已知的国内网站和国外网站区分开来, 国外网站则通过 dnscrypt-proxy 服务查询, 国内网站则继续使用运营商提供的 dns 服务.

``` bash
chinadns-ng -v -b 0.0.0.0 -l 53 -g /etc/chinadns/gfwlist.txt -m /etc/chinadns/chnlist.txt -c 国内dns服务IP#53 -t 127.0.0.1#5003
```

这样, 路由器上使用 chinadns-ng 来作为内网的域名查询服务, 其中参数 -c 表示用国内查询的 dns 服务地址与端口(通过 /etc/chinadns/chnlist.txt 列表确定)
其中参数 -t 表示用国外查询的 dns 服务地址与端口(通过 /etc/chinadns/gfwlist.txt 列表确定)

gfwlist.txt 和 chnlist.txt 文件在 chinadns-ng 的 github 仓库有提供.

至此, 内网所有机器均可自动代理科学上网, 并且不影响国内网络的使用.
