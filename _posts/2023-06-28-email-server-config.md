---
layout: post
title: 搭建自己的邮件服务
---

`Mailu.io` 是一个简单易用的邮件系统，使用它搭建邮件服务器十分简单，主要通过以下几个步骤就可以完成搭建。

首先通过 [https://setup.mailu.io/2.0/](https://setup.mailu.io/2.0/) 这个页面，创建配置文件，我的配置如下：

```bash
Mailu storage path: /mailu
Main mail domain and server display name: jackarain.org
Postmaster local part: admin
Authentication rate limit per IP for failed login: 5 hour
Authentication rate limit per user: 50 day
Outgoing message rate limit (per user): 200 day
Website name: Jackarain mail
Linked Website URL: https://jackarain.org
Enable the admin UI (and path to the admin UI): /admin
Enable Web email client (and path to the Web email client): roundcube /webmail (Enable the webdav service/Enable fetchmail/Enable oletools)
IPv4 listen address: 0.0.0.0
Subnet of the docker network: 192.168.203.0/24
Enable an internal DNS resolver (unbound) (.)
Public hostnames: jackarain.org
```

完成后，然后跳到页面，在服务器上按页面指示：

```bash
mkdir /mailu
cd /mailu
wget https://setup.mailu.io/2.0/file/70ff5570-9284-4696-b351-ca87281901dc/docker-compose.yml
wget https://setup.mailu.io/2.0/file/70ff5570-9284-4696-b351-ca87281901dc/mailu.env

cd /mailu
docker compose -p mailu up -d
docker compose -p mailu exec admin flask mailu admin admin jackarain.org PASSWORD
```

其中的 `PASSWORD` 替换为自己的密码，登陆用户名为 admin

此时服务器配置完成。

DNS 相关的配置：

添加 `A`、`AAAA` 记录 `mail.jackarain.org` 指向服务器

添加 `MX` @记录 `jackarain.org` 指向 `mail.jackarain.org` 并且设置权重，默认为 10

添加 `TXT` `_dmarc` 记录内容为：`"v=DMARC1; p=none; pct=100; rua=mailto:admin@jackarain.org"`

添加 `SPF` 条目 `TXT` @记录内容为：`"v=spf1 mx ~all"`

此时 `DNS` 相关配置完成，然后配置 `PTR` 记录，需要在 `VPS` 商的服务面板配置，找到配置 `PTR记录`
 的地方，将 `IP` 配置为 `mail.jackarain.org`

在服务正常运行后，邮件客户端可以使用 `IMAP`，收发服务器使用：`jackarain.org`
