---
title: 网络问题：从手搓wireguard（wsl，dgx）到认命用clash
date: 2026-02-02 02:30:10
tags: [tech, report]
category: tech
---


最初情况：pip直接下载环境，下载模型太慢，希望能直连外网加速模型下载。

最初的解决方式：通过wireguard，连接了digitalocean里租用的sfo的服务器，通过设置公钥私钥，建立wg quickup，初步连接了dgx机器和sfo的vps，并成功拉取了docker镜像以及一些翻墙才能拉去的包。

随后，我尝试通过不在lan的情况下用nvidia sync连接dgx（用我自己的mac/windows），发现不行，当时找到的办法是通过sfo vps当跳板机，通过mac-vps-dgx的方式远程连接dgx

第二天我尝试这样去连接dgx，但首先是发现dgx此时无法通过ssh连接到vps，同时似乎也无法ping谷歌。

随后我开始尝试各种方法都没成功，包括recovery。我在本地用wsl连接 vps也不成功。

感觉最好的方式还是直接用clash这种vpn。

# 更新：clash过于简单，几乎是一键修好（能在wsl用）



🔖 目标

让 DGX Ubuntu 机器能稳定 科学上网（访问 Docker Hub、pip、模型库等）。

💡 核心结论
✅ 让 DGX 出网正确的方法

必须让 DGX 的全局网络走 VPN 隧道（WireGuard / OpenVPN 等），不是依赖本机代理。

本地代理（如 Clash）只能让本机/WSL 出网，不能自动影响 DGX 的出口。

🧠 CLASH vs 商业 VPN vs 自建 VPS
方案	让 DGX 自己出网	复杂度	推荐
Clash 本机代理	❌ 不适合（只能对本机/WSL 生效）	低	✖
商业 VPN（WireGuard/OpenVPN）	✅ 可以	中	⭐⭐⭐⭐
自建 VPN（你自己的 VPS 作 WireGuard Server）	✅ 可以	高	⭐⭐⭐
📌 为什么 Clash 不能直接让 DGX 出网？

Clash 是客户端代理，设计是抓取本机/本地应用流量．

它产生的只是本机 SOCKS/HTTP 代理端口，不是系统级路由。

DGX 上的 apt/pip/docker 不会自动走该代理，除非手动在每个服务里配置，非常困难且不全面。

服务器级“全局科学上网”必须通过 VPN 隧道，实现内核层路由修改。

📌 商业 VPN 是否提供本机代理？

一些商业 VPN 客户端可能会提供本地代理端口（SOCKS/HTTP）作为附加功能。

但 商业 VPN 的核心能力是 VPN 隧道（OpenVPN / WireGuard）。

大多数 VPN 提供的是VPN 隧道出口，不是本机代理端口。

你需要确认所选的商业 VPN 是否真正提供 WireGuard/OpenVPN 配置可用于 Linux（特别是 DGX）。

📌 为什么 VPS 仍然有用

VPS 常见用途：

✔ 做 WireGuard VPN server（让 DGX 走这个出口）
✔ 做 NAT 出网出口（DGX 走 VPS 出网）
✔ 做跳板机（远程访问 DGX）

但如果你决定用 商业 VPN 直接让 DGX 出网，并且不再需要 VPS 做跳板/出口，那么：

👉 退订 VPS 是合理的（取决于你是否还需要它做别的事）。

🧪 推荐的工程级方法
📍 最推荐：商业 VPN + DGX WireGuard

选择支持标准 VPN 配置的商业 VPN（如 Mullvad / ProtonVPN / Surfshark 等）

获取 WireGuard/OpenVPN 配置

在 DGX 上安装 WireGuard/OpenVPN 客户端

配置允许所有流量（AllowedIPs = 0.0.0.0/0）

启动 VPN 隧道

验证 DGX 出网状态

🧠 验证 DGX 是否走 VPN 隧道有效

测试：

curl https://ipinfo.io


如果输出的是国外/VPN 提供商的 IP而非原 ISP IP，则 VPN 成功。

📌 备选：让 WSL 出网（Clash）

如果只是想让 Windows 或 WSL 出网翻墙：

✔ 安装 Clash 在 Windows
✔ 启用本地 SOCKS/HTTP 代理
✔ 在 WSL 里配置应用走代理
✔ 但这不会改变 DGX 的出口

🧠 核心工程原则（可长期复用）

服务器级出网必须用 VPN 隧道

本机代理只能影响本机

不要把代理软件当做 VPN

验证出口 IP 是确认科学上网成功的唯一方法

当你需要 DGX 自己访问互联网，必须配置系统级路由；；；

