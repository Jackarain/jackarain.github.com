---
layout: post
title: 基于 proxy_server TUN 模式的 OpenWrt 路由器透明代理指南
---

本指南介绍如何在 OpenWrt 路由器上单独使用 **proxy_server** 部署全局与策略透明代理。

通过运行原生集成的 **TUN 模式（TUN2SOCKS）** 及 **DoH（DNS over HTTPS）** 模块，无需依赖 `tun2socks`、`dnscrypt-proxy` 或 `chinadns-ng` 等第三方组件，即可实现内网设备透明上网与 DNS 防污染。

---

## 架构原理

```text
[ 内网设备 ] (手机/PC/电视)
     │
     ▼ (默认网关)
[ OpenWrt 路由器 ]
     │
     ├─ 域名/IP 命中规则 ──► [ proxy_server tun3 ] ──► (加密上游) ──► 境外网络
     │                                │ (so_mark=100)
     │                                ▼ (防环策略路由)
     │                           [ 物理 WAN 口 ]
     │                                │
     └─ 未命中规则 ─────────────────────┴─────────────► 本地/国内网络
```

* **IP/流量接管**：通过创建虚拟网卡 `tun3` 并配置路由表，将目标流量引入 `proxy_server` 的用户态协议栈。
* **策略分流**：根据 `proxy_domains`（域名后缀）与 `proxy_cidr`（IP 规则）进行双引擎判别。命中规则的流量走代理，未命中的流量通过物理网口直连。
* **防环路（Anti-Looping）**：`proxy_server` 自身向上游代理发起的连接会被赋予 `so_mark=100` 标记。通过 Linux 策略路由（`ip rule`）强制将带有该标记的数据包绕过 TUN 设备，由物理 WAN 口直接发出去，避免死循环。
* **DNS 机制**：内置 DoH 代理直接接管 DNS 查询。域名匹配代理列表时，通过加密通道向 remote/DoH 节点发起查询，彻底规避国内 DNS 污染。

---

## 一、 环境要求与准备工作

1. **路由器系统**：OpenWrt 19.07 及更高版本（必须具备 `ip-full` 命令支持）。
2. **硬件架构**：ARM64 / x86_64 / MIPS 等（已编译好对应平台的 `proxy_server` 二进制文件）。
3. **上游代理**：已准备好可用的 upstream 代理节点（支持 HTTP/HTTPS/SOCKS5 等协议）。

物理 WAN 网卡名称确认（本教程以 `pppoe-wan` 为例）：
在 OpenWrt 终端中运行以下命令确定当前外网网卡：

```bash
ip route show default | awk '{print $5}'
```

---

## 二、 配置文件编写

在路由器上创建配置目录：

```bash
mkdir -p /etc/proxy-server
```

创建配置文件 `/etc/proxy-server/server.conf`，写入以下内容：

```ini
# 监听端口与认证（无密码认证供内网使用）
server_listen=0.0.0.0:1080
auth_users=

# 上游代理地址 (请按实际情况替换协议、账号密码、域名/IP及端口)
proxy_pass=https://test:passwd@example.remote.com:443
proxy_pass_ssl=true

# TUN 模式相关参数
tun=true
tun_name=tun3
tun_mtu=1500

# 出站连接标记（用于防环策略路由）
so_mark=100

# 域名与 IP 分流规则 (命中规则走代理)
proxy_domains=google.com,youtube.com,telegram.org,github.com,twitter.com,facebook.com,instagram.com
proxy_cidr=103.235.0.0/16,117.168.0.0/16,149.154.0.0/16,185.199.0.0/16,91.108.0.0/16

# 加密 DNS 上游（防污染）亦可使用 proxy_pass 作为 DoH 节点
dns_upstream=https://dns.google/dns-query

# 代理功能开关
disable_udp=false
disable_logs=true
```

---

## 三、 路由与防环初始化脚本

创建脚本 `/etc/proxy-server/setup_routes.sh`：

```bash
#!/bin/sh

# 1. 定义变量
WAN_DEV="pppoe-wan"   # 替换为实际的物理 WAN 口名称
TUN_DEV="tun3"
TUN_IP="172.16.8.1/24"
MARK_ID="100"
TABLE_ID="100"

# 2. 获取 WAN 口默认网关
WAN_GW=$(ip route show default dev "$WAN_DEV" 2>/dev/null | awk '{print $3}')

if [ -z "$WAN_GW" ]; then
    echo "错误: 未能定位 $WAN_DEV 的网关 IP，请检查物理网络链接。"
    exit 1
fi

# 3. 创建防环策略路由（标记为 MARK_ID 的包走物理 WAN 接口网关直连）
ip route replace default via "$WAN_GW" dev "$WAN_DEV" table "$TABLE_ID" 2>/dev/null
ip rule del fwmark "$MARK_ID" table "$TABLE_ID" 2>/dev/null
ip rule add fwmark "$MARK_ID" priority 100 table "$TABLE_ID"

# 4. 配置并启用虚拟 TUN 网卡
ip addr add "$TUN_IP" dev "$TUN_DEV" 2>/dev/null
ip link set dev "$TUN_DEV" up 2>/dev/null

# 5. 设置全局流量路由到 TUN 设备 (使用折半子网掩码，避免覆盖 default 路由)
ip route add 0.0.0.0/1 dev "$TUN_DEV" 2>/dev/null
ip route add 128.0.0.0/1 dev "$TUN_DEV" 2>/dev/null

echo "TUN 防环策略路由与接口设置完成。"
```

赋予脚本可执行权限：

```bash
chmod +x /etc/proxy-server/setup_routes.sh
```

---

## 四、 创建 OpenWrt 系统服务 (init.d)

为了支持开机自启和后台服务化管理，创建 `/etc/init.d/proxy-server` 文件：

```bash
#!/bin/sh /etc/rc.common

START=99
STOP=10

USE_PROCD=1
PROG=/usr/bin/proxy_server
CONF=/etc/proxy-server/server.conf
ROUTE_SCRIPT=/etc/proxy-server/setup_routes.sh

start_service() {
    # 在服务启动前拉起 TUN 设备和策略路由
    procd_open_instance
    procd_set_param command $PROG --config $CONF
    procd_set_param respawn
    procd_set_param stdout 0
    procd_set_param stderr 0
    procd_close_instance

    # 延迟 2 秒等待 TUN 设备建立后加载路由
    ( sleep 2 && $ROUTE_SCRIPT ) &
}

stop_service() {
    # 清理路由规则
    ip rule del fwmark 100 table 100 2>/dev/null
    ip route del 0.0.0.0/1 dev tun3 2>/dev/null
    ip route del 128.0.0.0/1 dev tun3 2>/dev/null
}
```

赋予服务脚本执行权限并设置开机自启：

```bash
chmod +x /etc/init.d/proxy-server
/etc/init.d/proxy-server enable
```

---

## 五、 启动与状态校验

### 1. 手动启动服务

```bash
/etc/init.d/proxy-server start
```

### 2. 检查运行状态

* **检查进程状态**：

```bash
ps | grep proxy_server
```

* **检查 TUN 设备 IP 状态**：

```bash
ip addr show tun3
```

* 预估输出：包含 `inet 172.16.8.1/24 scope global tun3`。
* **检查策略路由规则**：

```bash
ip rule show | grep 100
```

* 预估输出：`100: from all fwmark 0x64 lookup 100*`
* **链路联通性测试（在内网设备或路由器终端执行）**：

```bash
curl -I https://www.google.com
```

若返回 HTTP 200 响应，说明代理分流及 DNS 模块功能均正常运作。
