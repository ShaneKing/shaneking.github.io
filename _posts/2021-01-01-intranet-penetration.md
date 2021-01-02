---
layout: post
title: 内网穿透
categories: Linux
description: 内网穿透
keywords: 内网穿透, NAT, Intranet penetration, nps
---


自从花生壳搞创收之后，基本上就是各种变着法的 get money，so 搭建搞起来。

## 需求
外网可远程家里的机器

## 分析
站在前辈的基础上，ngrok 私有了，frp 太粗糙，nps 也有 15.4k Star 了

![](/images/posts/2021/01/aHR0cHM6Ly9jNHlzLmNvbS93cC1jb250ZW50L3VwbG9hZHMvMjAxOS8wOC9pbWFnZS0xNTY3MDQ2MzY4NjQwLnBuZw.png)

## 设计
### 所需材料
- 一台公网机器（比如BAT云）
- 一个域名（通常域名服务商都提供解析服务）
- 火墙端口：40080，48024～48080

### 限制条件
- 内网机能与公网机互通 或 内网机能上外网但没有公网

## 开发
### 服务端
```bash
if ! [ -x "$(command -v nps)" ]; then
  mkdir -p ${SK_EXP__GIR_REPO_DIR}/ws/u20/ali/sh/o/nps && mv -f ${SK_EXP__GIR_REPO_DIR}/ws/u20/ali/sh/o/nps ${SK_EXP__GIR_REPO_DIR}/ws/u20/ali/sh/o/nps.skbak$(date +'%Y%m%d%H%M%S')
  mkdir -p ${SK_EXP__GIR_REPO_DIR}/ws/u20/ali/sh/o/nps && cd ${SK_EXP__GIR_REPO_DIR}/ws/u20/ali/sh/o/nps
  tar -xzvf ${SK_EXP__GIR_REPO_DIR}/u20/ali/sh/o/nps/linux_amd64_server.tar.gz -C .
  sudo ./nps install
  sed -i 's|http_proxy_port=80|http_proxy_port=40080|g' /etc/nps/conf/nps.conf
  sed -i 's|https_proxy_port=443|https_proxy_port=|g' /etc/nps/conf/nps.conf
  sed -i 's|bridge_port=8024|bridge_port=48024|g' /etc/nps/conf/nps.conf
  sed -i 's|public_vkey=123|public_vkey=${SK_EXP__NPS__PK}|g' /etc/nps/conf/nps.conf
  sed -i 's|web_host=a.o.com|web_host=web.nps.shaneking.org|g' /etc/nps/conf/nps.conf
  sed -i 's|web_password=123|web_password=${SK_EXP__NPS__PWD}|g' /etc/nps/conf/nps.conf
  sed -i 's|web_port = 8080|web_port = 48080|g' /etc/nps/conf/nps.conf
  sed -i 's|auth_crypt_key =1234567812345678|auth_crypt_key =${SK_EXP__NPS__ACK16}|g' /etc/nps/conf/nps.conf
  systemctl enable Nps
  systemctl restart Nps
  #systemctl status Nps

  sh $(dirname $0)/u20/ali/sh/i/nginx.sh
  cat >/etc/nginx/conf.d/nps.conf <<EOF
server {
    listen 80;
    server_name *.nps.shaneking.org;
    location / {
        proxy_set_header Host $host:$server_port;
        proxy_pass http://127.0.0.1:40080;
    }
}
server {
    listen 80;
    server_name web.nps.shaneking.org;
    location / {
        proxy_set_header Host $host:$server_port;
        proxy_pass http://127.0.0.1:48080;
    }
}
server {
    listen 443 ssl;
    server_name *.nps.shaneking.org;
    ssl_certificate  /etc/nps/conf/server.pem;
    ssl_certificate_key /etc/nps/conf/server.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        proxy_set_header Host $host:$server_port;
        proxy_pass http://127.0.0.1:40080;
    }
}
server {
    listen 443 ssl;
    server_name web.nps.shaneking.org;
    ssl_certificate  /etc/nps/conf/server.pem;
    ssl_certificate_key /etc/nps/conf/server.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        proxy_set_header Host $host:$server_port;
        proxy_pass http://127.0.0.1:48080;
    }
}
EOF

  nginx -s reload

  cd ${SK_EXP__GIR_REPO_DIR}
else
  echo 'nps installed.'
fi
```
### 管理端
![](/images/posts/2021/01/QQ20210101-132201@2x.png)
### 客户端
#### 树莓派
```bash
if ! [ -x "$(command -v npc)" ]; then
  mkdir -p ${SK_EXP__GIR_REPO_DIR}/ws/r32/o/npc && mv -f ${SK_EXP__GIR_REPO_DIR}/ws/r32/o/npc ${SK_EXP__GIR_REPO_DIR}/ws/r32/o/npc.skbak$(date +'%Y%m%d%H%M%S')
  mkdir -p ${SK_EXP__GIR_REPO_DIR}/ws/r32/o/npc && cd ${SK_EXP__GIR_REPO_DIR}/ws/r32/o/npc
  tar -xzvf ${SK_EXP__GIR_REPO_DIR}/r32/o/npc/linux_arm_v7_client.tar.gz -C .
  sudo mkdir -p /etc/npc/conf && sudo chmod -R 777 /etc/npc
  cat >/etc/npc/conf/npc.conf <<EOF
[common]
server_addr=nps.shaneking.org:48024
conn_type=tcp
vkey=${SK_EXP_RPI__NPC__VK}
auto_reconnection=true
max_conn=1000
flow_limit=1000
rate_limit=1000
#basic_username=11
#basic_password=3
#web_username=user
#web_password=1234
crypt=true
compress=true
#pprof_addr=0.0.0.0:9999
disconnect_timeout=60

[tcp22]
mode=tcp
target_addr=127.0.0.1:22
server_port=48025

[tcp5900]
mode=tcp
target_addr=127.0.0.1:5900
server_port=48026
EOF

  sudo ./npc install -config=/etc/npc/conf/npc.conf
  sudo systemctl enable Npc
  sudo systemctl restart Npc
  #sudo systemctl status Npc

  cd ${SK_EXP__GIR_REPO_DIR}
else
  echo 'npc installed.'
fi
```
## 验证
### 服务端
![](/images/posts/2021/01/QQ20210101-000557@2x.png)
### 客户端
#### ssh
```bash
ShaneKing@ShaneKing-MBP13R2012 Downloads % ssh -p 48025 pi@nps.shaneking.org
The authenticity of host '[nps.shaneking.org]:48025 ([106.14.198.173]:48025)' can't be established.
ECDSA key fingerprint is SHA256:aedq/iqM4eWrTwoKdYCxpk8J0aWke4/QLjNXGQA0A0A.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[nps.shaneking.org]:48025,[106.14.198.173]:48025' (ECDSA) to the list of known hosts.
pi@nps.shaneking.org's password:
Linux raspberrypi 5.4.79-v7l+ #1373 SMP Mon Nov 23 13:27:40 GMT 2020 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jan  1 12:52:34 2021
pi@raspberrypi:~ $
```
#### vnc
![](/images/posts/2021/01/QQ20210101-131304@2x.png)

## 参考
<https://blog.csdn.net/a1035434631/article/details/108010819>