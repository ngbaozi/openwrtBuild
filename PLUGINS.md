# 最终插件清单与取舍说明

> 定稿 2026-11-11（20:18 追加：去广告从 AdGuardHome 换成 `adblock-fast`）。所有包名与依赖均比对过 `coolsnowwolf/luci@openwrt-25.12`（14205 项 / 180 个应用）、`coolsnowwolf/packages`（7935 项）、`fw876/helloworld`（1121 项）。**不使用任何第三方 feed，不存在同名包被覆盖的问题。** 完整改动列表见 `CHANGES.md`。

## 一、最终编入的插件

| 分组 | 包 | 作用 |
|---|---|---|
| 代理 | `luci-app-ssr-plus` + `xray-core`、`mihomo`、`v2ray-core`、`sing-box` | 唯一透明代理面板；Xray 覆盖 VLESS Reality / XTLS / xhttp，Mihomo 覆盖 Clash 系节点 |
| DNS / 去广告 | `luci-app-adblock-fast` + `adblock-fast`、`luci-app-mosdns` | **adblock-fast 在 dnsmasq 层拦广告**（出厂已启用），MosDNS 做国内外分流；两者不争抢 53 |
| 多 WAN | `luci-app-mwan3`、`luci-app-syncdial` | 多 WAN 负载均衡 + 单线多拨（需运营商支持） |
| 文件 / 娱乐 | `luci-app-filebrowser`、`luci-app-uugamebooster`、`luci-app-unblockneteasemusic` | 网页文件管理 / UU 加速 / 网易云解锁 |
| 主题 | `luci-theme-argon`、`luci-theme-design`、`luci-app-argon-config`、`luci-app-design-config` | 两套主题 + 各自设置页，运行时切换 |
| 基础运维 | `luci-app-filetransfer`、`ttyd`、`diskman`、`nlbwmon`、`watchcat`、`oui`、`sysctl`、`arpbind`、`wol`、`timecontrol` | 网页装 ipk / 网页终端 / 磁盘管理 / 客户端流量 / 断网自愈 / 厂商图标 / 内核参数 / IP-MAC 绑定 / 网络唤醒 / 上网时段 |
| 远程访问 | `luci-app-tailscale-community`、`luci-app-lucky` | 异地组网；DDNS + 端口转发 + 反向代理 + 证书 |
| 网络质量 | `luci-app-sqm`、`upnp`、`banip`、`irqbalance`、`attendedsysupgrade` | SQM 抗 bufferbloat / UPnP / 封禁恶意 IP / 中断均衡 / 在线升级 |
| 网络加速 ACC | `kmod-tcp-bbr`、`kmod-nft-offload`、`kmod-nft-fullcone` + `files/` 出厂默认值 | BBR、FullCone NAT 默认开；软件 Flow Offload 默认关（见下）。开关全部在 LuCI 防火墙页面，无独立面板 |
| 重量级 | `luci-app-dockerman` + `dockerd`、`docker`、`luci-app-openlist`、`luci-app-qbittorrent`、`luci-app-syncthing` | Docker / 网盘挂载 / BT 下载 / 文件同步。`dockerman` 不自动拉服务，故显式补 `dockerd` + `docker` |

`config.seed` 里另有 9 个 ssr-plus 选项：透明代理后端选 nftables，并显式开启 Xray、Shadowsocks-Rust、Mihomo、GeoData、ChinaDNS-NG、MosDNS、Shadow-TLS、HTTP 代理。**其中 `INCLUDE_Xray` 最关键**：上游默认是 `NONE_V2RAY`，不开就没有 Reality / VLESS。

## 二、去重逻辑（这次砍掉了什么）

| 砍掉 | 原因 |
|---|---|
| `luci-app-passwall` | 与 ssr-plus 同为透明代理面板，同时启用会争抢 nftables 规则与 53 端口；协议覆盖三家都全 |
| `haproxy` | 原为 PassWall 的负载均衡依赖，面板移除后无使用者 |
| `passwall2` / `nikki` / `openclash` / `homeproxy` | 同类面板，一律不加 |
| `ddns-go` / `frpc` / `natmap` / `zerotier` | `lucky` 已含 DDNS 与端口转发，`tailscale` 已覆盖异地组网 |
| `transmission` | 与 `qbittorrent` 同为 BT 客户端 |
| `vnstat2` / `netdata` | 与 `nlbwmon` 重复或体积过大 |
| `luci-app-adguardhome`（20:18 追加删除） | Go 常驻进程、独立监听 53，与 MosDNS 需手动串联才不解析循环；改用官方轻量 `adblock-fast` |
| `adblock` / `simple-adblock` | 经典 adblock 依赖 `firewall4` + `coreutils` + `gawk` 更重；`simple-adblock` 在官方 feed 里没有 LuCI 面板 |
| `luci-app-turboacc` / `turboacc-config` | 不在任何官方 feed（只在 `kenzok8/small-package/other/lean/`）；它是 iptables / `xt_FLOWOFFLOAD` + `kmod-ipt-offload` 时代的旧面板，与 25.12 的 nftables / `kmod-nft-offload` 不兼容。官方 LuCI 已把 BBR、FullCone NAT、Flow Offload 三个开关并入防火墙页面 |
| `luci-app-xlnetacc` | 迅雷快鸟宽带提速，需账号且仅特定运营商可用，不是网络加速 |
| `cpufreq` | 底层包 `cpufreq` 不在官方 packages feed，写了会被 defconfig 静默丢弃 |
| `msd_lite` / `udpxy` | 无 IPTV 需求，不占编译时间 |
| 两个 kenzok8 feed | 与官方 feed 十几个同名包，来源不可控；官方依赖已齐全 |

**去广告与分流的关系（换了方案后变简单了）**：`adblock-fast` 不监听任何端口，只是把黑名单以 `dnsmasq.servers` 形式写进 dnsmasq 配置，处理完就退出、不常驻内存；MosDNS 继续做国内外分流。**刷机后不需要再手工串联两个 53 端口进程**，也不存在原来的解析循环风险。唯一外部依赖是规则源下载，出厂已预置两个不同 CDN 的规则，任一可达即生效。

## 三、来源为何收敛到官方

- `coolsnowwolf/packages` 已含 `adblock-fast`、`ddns-go`、`lucky`、`openlist`、`dockerd`、`docker`、`ttyd`、`watchcat`、`nlbwmon`、`banip`、`sqm-scripts`、`irqbalance`、`netdata`、`geo2txt`、`v2ray-geodata`、`sing-box`、`xray-core`、`tailscale`、`qBittorrent`、`syncthing`、`UnblockNeteaseMusic`、`uugamebooster`、`filebrowser`、`mwan3` 等全部底层依赖。
- 官方 golang 是 **1.27.3**，kenzok8 README 里「换成 golang 1.26」已过时。
- `helloworld` 提到 `feeds.conf` 首行后，其 `xray-core` 的 `eip93/overlay.py` 与 `190-an7581-eip93-aes-gcm.patch`、`mihomo` 的 `100-an7581-eip93-aes-gcm.patch` 才会被选中；否则会被官方 packages 的同名上游版本覆盖，硬件加速补丁就没了。

## 四、使用与风险

1. **只启用一个透明代理**。当前固件里只有 ssr-plus，以后若加了第二个面板，务必只开一个服务。
2. **去广告出厂已开**：`files/etc/uci-defaults/98-adblock-fast` 在首次开机把 `enabled` 置 1 并启用 Hagezi Pro / OISD Big 两个规则源。官方包默认全是关的，不注入就等于空面板。在「服务 → AdBlock-Fast」可换源、加白名单、暂停；若国内 CDN 拉取失败，换成任一可访问的域名列表即可。
3. **ACC 默认组合**：BBR + FullCone NAT 开，软件 Flow Offload 关（与 SSR-Plus 透明代理、SQM 互斥，强开会出现「测速正常但分流失效」）。不用代理 / SQM 时在「网络 → 防火墙 → 区域设置」把 *Flow offloading type* 改成 `Software flow offloading` 即可，无需重编译。x86/64 没有 HNAT / SFE / NSS 等硬件加速，硬件卸载保持关闭。
4. **首次编译 4–6 小时**（十几个 Go 程序 + Docker 内核模块；换掉 AdGuardHome 后少一个 Go 常驻程序）。`timeout-minutes: 345` 是上限，缓存对第二次起效。若超时，先注释 `dockerman` / `dockerd` / `docker` 三行。
5. 两个主题都编进去了，在「系统 → 语言与界面」里切换，不会冲突。
6. 产物是 **ipk**，别混装 apk 第三方包。
