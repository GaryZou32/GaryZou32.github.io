---
title: 通过vps连接linux和外网
date: 2026-02-01 13:47:42
tags: [tech]
category: tech
---

今日关键词：clawdbot，nips，digitalocean，talescale, WireGuard

Linux 上怎么配置代理或 VPN 来访问外网

答:
租用海外vps, 然后用wireguard连接
选了digital ocean的sfo节点，选用8美元的premium intel套餐
自己搭建WireGuard vpn

在VPS端：
1. 安装WireGuard
2. 生成服务器密钥 sudo wg genkey | sudo tee /etc/wireguard/server.key | sudo wg pubkey | sudo tee /etc/wireguard/server.pub
3. 写 Server 配置 /etc/wireguard/wg0.conf
内容类似
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <VPS 私钥>

PostUp   = sysctl -w net.ipv4.ip_forward=1 && iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <DGX 公钥>
AllowedIPs = 10.10.0.2/32

4. 启动WireGuard Server
sudo wg-quick up wg0
sudo wg show

5. 主机端：
同样，安装wire guard，生成客户端密钥，把client pub添加到VPS的Peer配置
写DGX的客户端配置
[Interface]
Address = 10.10.0.2/24
PrivateKey = <DGX 私钥>
DNS = 1.1.1.1

[Peer]
PublicKey = <VPS 公钥>
Endpoint = <VPS 公网 IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25

6. 启动客户端 sudo wg-quick up wg0

最终测试curl -I --connect-timeout 8 https://registry-1.docker.io/v2/


