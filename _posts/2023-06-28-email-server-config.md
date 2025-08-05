---
layout: post
title: 搭建自己的邮件服务
---

# 邮件服务器配置文档

## 创建 docker-compose.yml 文件

在 `/opt/mailserver` 目录下创建 `docker-compose.yml` 文件，内容如下：

```yaml
services:
  mailserver:
    image: docker.io/mailserver/docker-mailserver:latest
    hostname: mail.jackarain.org
    domainname: mail.jackarain.org # 改为你的域名
    container_name: mailserver
    ports:
      - "25:25"    # SMTP
      - "143:143"  # IMAP
      - "465:465"  # ESMTP (implicit TLS)
      - "587:587"  # Submission (TLS)
      - "993:993"  # IMAPS
    volumes:
      - ./docker-data/dms/mail-data/:/var/mail/
      - ./docker-data/dms/mail-state/:/var/mail-state/
      - ./docker-data/dms/mail-logs/:/var/log/mail/
      - ./docker-data/dms/config/:/tmp/docker-mailserver/
      - ./docker-data/certbot/certs/:/etc/letsencrypt/:ro
      - /etc/localtime:/etc/localtime:ro
    restart: always
    stop_grace_period: 1m
    environment:
      - ENABLE_SPAMASSASSIN=1
      #- ENABLE_CLAMAV=1
      #- ENABLE_FAIL2BAN=1
      - SSL_TYPE=letsencrypt
      - PERMIT_DOCKER=host
      - ONE_DIR=1
      - DMS_DEBUG=0
      - POSTFIX_MESSAGE_SIZE_LIMIT=152428800 # 附件大小
    cap_add:
      - NET_ADMIN
      - SYS_PTRACE
  certbot-cloudflare:
    image: certbot/dns-cloudflare:latest
    command: certonly --dns-cloudflare --dns-cloudflare-credentials /run/secrets/cloudflare-api-token -d mail.jackarain.org --non-interactive --agree-tos --email jack.wgm@gmail.com
    volumes:
      - ./docker-data/certbot/certs/:/etc/letsencrypt/
      - ./docker-data/certbot/logs/:/var/log/letsencrypt/
    secrets:
      - cloudflare-api-token
secrets:
  cloudflare-api-token:
    file: ./secrets/cloudflare.ini # 指向本地文件
```

## 创建 Cloudflare API 密钥文件

在 `/opt/mailserver` 目录下创建 `secrets/cloudflare.ini` 文件，内容为 Cloudflare 的 DNS API 密钥，格式如下：

```ini
dns_cloudflare_api_token = 你的API令牌
```

## 启动 Docker 服务

```bash
docker-compose pull
docker-compose up -d
```

若需要停止，则可以

```bash
docker-compose down
```

## 生成 DKIM 密钥

执行以下命令生成 DKIM 密钥：

```bash
docker exec -ti mailserver setup config dkim domain jackarain.org
```

## 获取 DKIM 公钥

执行以下命令获取 DKIM 公钥：

```bash
cat ./docker-data/dms/config/opendkim/keys/jackarain.org/mail.txt
```

输出示例（实际内容可能不同）：

```txt
mail._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
"p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtaWCaddumYV/T0qsMe4/R7KErQtH7/0rXfS1GNotflFzW/ibS68MLzQL8jFn8v87+n0DTt0emOEYRj9rU8MW2600wOgVYUd9QododS5BIscqPt9t2PhgdyPBTwsgu3ewjzKhQb01pTtFFo9nHFX2MO93KaAB/1KKwZCBBqoWjYbF+ehoU4TM6S0rDd5HvogdmuG20aVyg/bs87"
"5VRP35PLiYJVeHrouBk3JvSVVgx93bajXkINZj7YCeRZnMU9vyJuGUABpAcwtcWBToSqJzJsw4Htbpcu4xg32JzP46EsqDQwoEE7h8q6UXkMuzZu9bzkFQ5fTgCuvx6F+2hvQwFwIDAQAB" ) ; ----- DKIM key mail for jackarain.org
```

## 配置 DNS 记录

### A 和 AAAA 记录

- 添加 `A` 记录：`mail.jackarain.org` 指向服务器 IPv4 地址。
- 添加 `AAAA` 记录：`mail.jackarain.org` 指向服务器 IPv6 地址（如果有）。

### MX 记录

- 添加 `MX` 记录：`@` 记录指向 `mail.jackarain.org`，优先级默认为 `10`。

### DKIM 记录

- 添加 `TXT` 记录：名称为 `mail._domainkey.jackarain.org`（注意：有些DNS服务商只需要填写`mail._domainkey`），内容为：

```txt
  v=DKIM1; h=sha256; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtaWCaddumYV/T0qsMe4/R7KErQtH7/0rXfS1GNotflFzW/ibS68MLzQL8jFn8v87+n0DTt0emOEYRj9rU8MW2600wOgVYUd9QododS5BIscqPt9t2PhgdyPBTwsgu3ewjzKhQb01pTtFFo9nHFX2MO93KaAB/1KKwZCBBqoWjYbF+ehoU4TM6S0rDd5HvogdmuG20aVyg/bs875VRP35PLiYJVeHrouBk3JvSVVgx93bajXkINZj7YCeRZnMU9vyJuGUABpAcwtcWBToSqJzJsw4Htbpcu4xg32JzP46EsqDQwoEE7h8q6UXkMuzZu9bzkFQ5fTgCuvx6F+2hvQwFwIDAQAB
```

  （注意：实际内容为`mail.txt`中的`TXT`记录值，去掉引号和括号，并合并`p`的值。但实际使用中，通常直接复制`mail.txt`中的内容，去掉换行和引号，然后作为TXT记录值。具体格式请根据DNS服务商调整）

### DMARC 记录

- 添加 `TXT` 记录：名称为 `_dmarc.jackarain.org`（有些DNS服务商只需要填写`_dmarc`），内容为：

  ```txt
  v=DMARC1; p=none; pct=100; rua=mailto:admin@jackarain.org
  ```

### SPF 记录

- 添加 `TXT` 记录：名称为 `@`（即根域名），内容为：

  ```txt
  v=spf1 mx ~all
  ```

## 配置 PTR 记录

在 VPS 服务商的面板中配置 PTR 记录（反向解析），将服务器的 IP 地址反向解析为 `mail.jackarain.org`。

## 创建邮箱用户

执行以下命令创建用户：

```bash
docker exec -ti mailserver setup email add root@jackarain.org
docker exec -ti mailserver setup email add postmaster@jackarain.org
docker exec -ti mailserver setup email add admin@jackarain.org
```

## 邮件客户端配置

- **接收邮件服务器 (IMAP)**: `mail.jackarain.org`
- **发送邮件服务器 (SMTP)**: `mail.jackarain.org`
- **用户名**: `root@jackarain.org`（或其他创建的用户）
- **端口**:
  - IMAP: 993 (SSL/TLS)
  - SMTP: 465 (SSL/TLS) 或 587 (STARTTLS)
- **使用安全连接**: SSL/TLS

至此，邮件服务器已配置完成并可以正常使用。
