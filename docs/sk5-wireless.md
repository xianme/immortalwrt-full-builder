# 无线 AP + 单设备单 IP + SK5 出口

这套功能解决的问题：**一台 x86 软路由开出 WiFi，接入的每台设备各自固定一个内网 IP，并各自走一条独立的 SK5(SOCKS5) 出口，一机一 IP。**

没有面板、没有数据库，全部用 UCI + POSIX shell + mihomo 实现，开机自启、掉线自愈。

---

## 1. 它包含什么

| 能力 | 实现方式 |
| --- | --- |
| 无线 AP | `wpad-basic-mbedtls` + 常见 x86 无线网卡驱动（Intel AX/BE、MTK、Atheros、Realtek），首次开机自动建 AP 并桥接进 `br-lan` |
| 单设备单 IP | 设备登记后自动分配 `LAN.10 - LAN.99` 的固定地址，写入 dnsmasq 静态租约；动态池固定在 `.100` 起，互不冲突 |
| 一机一出口 | mihomo `SRC-IP-CIDR` 规则：设备源 IP → 该设备自己的 SK5 代理，设备完全不用设代理 |
| 透明接管 | nftables TPROXY（TCP + UDP）把 `br-lan` 客户端流量导到 mihomo 的 `7895` 端口 |
| 防私改 IP | 可选 `ether saddr` + `ip saddr` 强制绑定，设备把自己改成别人的 IP 会直接被丢弃 |
| 防 IPv6 泄漏 | 默认关闭 LAN 的 IPv6 下发（`dhcpv6`/`ra`/`ndp`），避免设备用 IPv6 绕过代理暴露真实地址 |

---

## 2. 首次开机

1. 刷入固件，网线接 LAN 口，浏览器打开 `http://192.168.x.1/`（或 `sk5ctl status` 看 LAN 地址）。
2. 无线网卡被识别后会自动出现 AP：**SSID `SK5-WiFi`**，密码随机生成。
3. SSH 登录后执行 `sk5ctl status` 查看 SSID 与密码；也可以自己改：

   ```sh
   uci set sk5.wifi.ssid='我的WiFi'
   uci set sk5.wifi.key='至少8位密码'
   uci commit sk5
   sk5ctl ap          # 重新生成 AP 配置
   ```

> **注意**：x86 机器和虚拟机默认没有无线硬件。没有网卡时不会凭空出现 `wlan0`，
> 驱动只是保证插上常见网卡就能用。USB 网卡可先 `lspci -nn | grep -i network` 或 `lsusb` 确认型号。

---

## 3. 给设备绑定 SK5（核心用法）

一条命令登记一台设备：

```sh
sk5ctl add 手机01 socks5://用户名:密码@1.2.3.4:1080 AA:BB:CC:DD:EE:01
```

- `手机01`：设备名称（字母、数字、`.`、`_`、`-`）
- `socks5://...`：SK5 代理地址，也支持 `1.2.3.4:1080:用户名:密码` 和纯 `1.2.3.4:1080`
- `AA:BB:...`：设备的 WiFi MAC（可省略；省略时只按 IP 分流，没有静态租约）
- 第四个参数可指定 IP（省略则自动在 `.10-.99` 里挑一个）

设备连上 WiFi 后会拿到这个固定 IP，流量自动从这条 SK5 出去。查出口 IP：

```sh
# 在设备上执行
curl https://api.ipify.org
```

### 常用命令

```sh
sk5ctl list                          # 设备清单（名称/MAC/IP/状态/出口）
sk5ctl status                        # 服务、WiFi、LAN、端口概况
sk5ctl set 手机01 proxy socks5://u:p@5.6.7.8:1080
sk5ctl set 手机01 enabled 0          # 暂停该设备的代理（走直连）
sk5ctl set 手机01 enabled 1
sk5ctl set 手机01 ip 192.168.1.30    # 改固定 IP
sk5ctl set 手机01 mac AA:BB:CC:DD:EE:02
sk5ctl del 手机01
sk5ctl apply                         # 手工改过 uci 后重新生成并生效
sk5ctl wifi                          # 查看 SSID / 密码
sk5ctl ap                            # 插上无线网卡后补建 AP
```

---

## 4. 配置项

主配置 `/etc/config/sk5`：

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| `sk5.main.enabled` | `1` | 总开关 |
| `sk5.main.default_action` | `direct` | 未登记设备：`direct` 直连 / `block` 拒绝 |
| `sk5.main.enforce_mac_ip` | `1` | 强制 MAC/IP 绑定，防私改 IP |
| `sk5.main.disable_ipv6` | `1` | 关闭 LAN IPv6 下发 |
| `sk5.main.tproxy_port` | `7895` | TPROXY 入口端口 |
| `sk5.main.mixed_port` | `7890` | 局域网可直接用的 HTTP/SOCKS5 混合端口 |
| `sk5.wifi.ssid` / `key` | `SK5-WiFi` / 随机 | AP 名称与密码 |
| `sk5.wifi.country` | `CN` | 无线国家码 |

每台设备是 `config device` 段：`name`、`mac`、`ip`、`proxy`、`enabled`。

---

## 5. 工作原理

```
设备(固定IP) --WiFi--> br-lan
                         |
                  nft table inet sk5 (prerouting, mangle)
                    ├─ MAC/IP 不符 -> drop
                    ├─ 内网 / 回环 / SK5服务器地址 -> 直连
                    └─ 其余 TCP/UDP -> TPROXY :7895
                         |
                     mihomo
                    ├─ SRC-IP-CIDR,192.168.1.10/32 -> SK5-A
                    ├─ SRC-IP-CIDR,192.168.1.11/32 -> SK5-B
                    └─ MATCH -> DIRECT
```

- 固定 IP 来自 dnsmasq 静态租约（`/etc/config/dhcp` 里的 `sk5_*` 段，由 `sk5ctl` 自动维护）。
- 代理服务器自身地址会解析后放进 nft 的 `bypass` 集合，避免回环。
- TLS/HTTP/QUIC 开了 sniffer，并把域名回填给 SOCKS5，让代理侧做远端 DNS，减少 DNS 泄漏。
- 服务由 procd 托管（`/etc/init.d/sk5`），进程挂了自动拉起。

---

## 6. 常见问题

**没有无线信号？**
`iw dev` 看不到接口，说明机器没有无线网卡或驱动未覆盖。先 `lspci -nn | grep -i network`、`lsusb` 看型号；USB 网卡一般需要 `kmod-rtl8xxxu`、`kmod-rtw88-*` 等，重新 `make menuconfig` 加驱动即可。

**设备没走代理？**
`sk5ctl list` 确认设备已启用且 IP 正确；设备上 `ip addr` 看是否拿到静态地址（租约变化后需要重新连一次 WiFi 或 `renew`）。再用 `curl -x http://路由器IP:7890 https://api.ipify.org` 验证 mihomo 出口。

**SK5 连不上？**
`logread | grep mihomo` 看报错；先确认服务器地址、端口、账号密码，以及服务器是否允许你的出口 IP。UDP 不通时把代理的 `udp: true` 保留但注意部分 SK5 服务器不支持 UDP，此时只有 TCP 走代理。

**会不会 DNS 泄漏？**
设备 DNS 由路由器 dnsmasq 解析（走 WAN）。如果要求 DNS 也走代理，把设备 DNS 改成 `路由器IP` 并通过 mihomo 的 `redir-host`/DNS 接管另行配置；当前实现优先保证出口 IP 正确与稳定，避免 fake-ip 与 dnsmasq 冲突。

**IPv6 被关了，能开吗？**
`uci set sk5.main.disable_ipv6='0'; uci commit sk5; sk5ctl defaults` 后重启网络。开启后 IPv6 流量不经过 SK5，会暴露真实 IPv6 地址，工作室场景不建议开。

**能不用透明代理，手动设代理吗？**
可以：设备代理设为 `路由器IP:7890`（混合端口，HTTP/SOCKS5 都支持）。

---

## 7. 安全提醒

- `sk5ctl` 没有鉴权，只适合内网使用；不要把 `7890`/`9090` 暴露到公网。
- AP 默认 WPA2 + 随机密码，密码保存在 `/etc/config/sk5`（root 可读）。
- 设备名称、MAC、SK5 账号都写在 `/etc/config/sk5`，备份固件配置时注意别外泄。
