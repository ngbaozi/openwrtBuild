# 本次改动列表（2026-11-11）

**目标**：所有插件一次编进固件，后续不再 opkg 安装；消除同名包重复；来源收敛到官方 feed，保证可编译。

**结果**：编译包从 **13 个 CONFIG_PACKAGE 符号 → 51 个**（其中 LuCI 应用/主题 33 个、底层服务 6 个、ACC 加速内核模块 3 个、ssr-plus 选项 9 个）；第三方 feed 从 2 个 → 0 个；重复代理面板从 2 个 → 1 个。

---

## 一、插件来源改造（核心）

| 项 | 改前 | 改后 | 原因 |
|---|---|---|---|
| `kenzok8/openwrt-packages` | 插入第 1 行 | **移除** | 与官方 luci feed 有 `adguardhome`、`filebrowser`、`ddns-go`、`lucky`、`eqos`、`dockerman`、`smartdns`、`design-config`、`argon-config` 等大量同名包 |
| `kenzok8/small` | 插入第 2 行 | **移除** | 与 helloworld / 官方 luci feed 有 `ssr-plus`、`passwall`、`passwall2`、`openclash`、`homeproxy`、`nikki`、`xray-core` 等大量同名包 |
| `fw876/helloworld` | 被注释掉，用 kenzok8 整合版替代 | **恢复并提到 `feeds.conf` 首行** | Lean 官方维护；其 `xray-core` 带 `eip93/overlay.py` 硬件 AES 加速与 AnyTLS / TUIC / Snell / ShadowTLS / Kcptun 补丁，`mihomo` 带 `100-an7581-eip93-aes-gcm.patch`。OpenWrt `feeds install` 按 conf 顺序取先出现的同名包，提到首行才能选中这些版本 |
| 「删除重复包」命令 | `rm -rf feeds/luci/applications/luci-app-mosdns`、`feeds/packages/net/{alist,adguardhome,mosdns,xray*,v2ray*,sing*,smartdns}` 等 | **移除** | 那是给 kenzok8 版本让路的，现在没有第三方 feed，删了反而会破坏官方依赖 |
| `golang 1.26` 替换 | `git clone kenzok8/golang -b 1.26` | **移除** | 官方 `coolsnowwolf/packages` 的 golang 已是 **1.27.3**，比 1.26 更新；原做法已过时 |
| `feed_mode` 开关 | 手动触发时二选一 | **删除** | 只剩一套来源，开关无意义 |

> 可行性核实：拉取 `coolsnowwolf/packages` 完整 git 树（7935 项）逐一比对，官方 feed 已含 `ddns-go`、`lucky`、`openlist`、`dockerd`、`docker`、`ttyd`、`watchcat`、`nlbwmon`、`banip`、`sqm-scripts`、`irqbalance`、`netdata`、`geo2txt`、`v2ray-geodata`、`sing-box`、`xray-core`、`tailscale`、`zerotier`、`qBittorrent`、`transmission`、`syncthing`、`msd_lite`、`udpxy`、`UnblockNeteaseMusic`、`uugamebooster`、`filebrowser`、`mwan3` 等全部底层依赖。

## 二、删除的插件 / 符号

| 删除项 | 原因 |
|---|---|
| `luci-app-passwall` | 与 `ssr-plus` 同为透明代理面板，同时启用会争抢 nftables 规则与 53 端口；协议三家都全，留官方同源的 ssr-plus |
| `haproxy` | 原为 PassWall 负载均衡依赖，面板移除后无使用者 |
| 6 个失效符号（`adbyby-plus`、`turboacc`、`turboacc-config`、`ap-modem`、`broadbandacc`、`fastnet`） | 上一轮已删，feed 中不存在，defconfig 会静默丢弃 |

**保留但需串联**（不是重复）：`adguardhome`（广告过滤）+ `mosdns`（国内外分流）。二者分工不同，装好后须明确一条链路、只让一个进程监听 53。

## 三、新增插件（24 个 LuCI 应用/主题 + 4 个底层服务 + 3 个 ACC 内核模块 + 9 个 ssr-plus 选项）

| 分组 | 新增项 | 作用 |
|---|---|---|
| **基础运维（10）** | `filetransfer`、`ttyd`、`diskman`、`nlbwmon`、`watchcat`、`oui`、`sysctl`、`arpbind`、`wol`、`timecontrol` | 网页装包 / 网页终端 / 磁盘管理 / 客户端流量 / 断网自愈 / 厂商图标 / 内核参数 / IP-MAC 绑定 / 网络唤醒 / 时段控制 |
| **远程访问（2）** | `tailscale-community`、`lucky` | 异地组网；DDNS + 端口转发 + 反向代理 + 证书。因 lucky 已含 DDNS，**不再加** `ddns-go`、`frpc`、`natmap`、`zerotier` |
| **网络质量（5）** | `sqm`、`upnp`、`banip`、`irqbalance`、`attendedsysupgrade` | 抗 bufferbloat / UPnP（自动拉 `miniupnpd-nftables`）/ 恶意 IP 封禁 / 多核中断均衡 / 在线升级 |
| **重量级（4）** | `dockerman`、`openlist`、`qbittorrent`、`syncthing` | Docker 管理 / 多网盘挂载 / BT 下载 / 文件同步。加 `transmission` 会与 qBittorrent 重复，故不加 |
| **主题（3）** | `luci-theme-design`、`argon-config`、`design-config` | Design 主题（11-08 起上游默认）+ 两套主题的设置页；两种主题风格不同，运行时切换 |
| **底层服务（4）** | `xray-core`、`mihomo`、`dockerd`、`docker` | `dockerman` 的 Makefile 不带 `+dockerd`，必须显式补上服务与 CLI；xray/mihomo 兜底确保内核一定编入 |
| **ssr-plus 选项（9）** | `Nftables_Transparent_Proxy`、`INCLUDE_Xray`、`Shadowsocks_Rust_Client`、`Mihomo`、`GeoData`、`ChinaDNS_NG`、`MosDNS`、`Shadow_TLS`、`Http_Proxy` | 默认 `V2ray-core Selection = None`，**不显式开 `INCLUDE_Xray` 就没有 Reality / VLESS**；其余把常用协议与 DNS 组件一次开齐 |
| **ACC 内核模块（3）** | `kmod-tcp-bbr`、`kmod-nft-offload`、`kmod-nft-fullcone` | BBR / 软件 Flow Offload / nftables FullCone NAT。官方 `firewall4` 的 Makefile 已把它们列为 `DEPENDS`，显式写 `=y` 是兜底，确保一定编入 |

## 四、保留的插件（9 个 LuCI + 2 个底层）

`ssr-plus`、`adguardhome`、`mosdns`、`filebrowser`、`mwan3`、`syncdial`、`uugamebooster`、`unblockneteasemusic`、`luci-theme-argon`；底层 `v2ray-core`、`sing-box`。

## 五、网络加速（ACC）专项

**结论：不需要 `luci-app-turboacc`，官方 feed 里本来就有同等能力。** 核实过程：

- `coolsnowwolf/lede@master` 的 `package/network/config/firewall4/Makefile` 中，`Package/firewall4` 的 `DEPENDS` 已含 `+kmod-tcp-bbr +kmod-nft-offload +kmod-nft-fullcone`。其中 `kmod-tcp-bbr` 定义在 `package/kernel/linux/modules/netsupport.mk` 并自带 `/etc/sysctl.d/12-tcp-bbr.conf`（写入 `net.ipv4.tcp_congestion_control=bbr`），`kmod-nft-offload` 在 `netfilter.mk`，`kmod-nft-fullcone` 在 `package/network/services/fullconenat-nft/Makefile`。
- `coolsnowwolf/luci@openwrt-25.12` 的 `luci-app-firewall/.../view/firewall/zones.js` 已内置 `fullcone`、`fullcone6`、`tcpcca` 选项以及 `Flow offloading type`（None / Software / Hardware / MTK HNAT / QCA ECM），并自带 "Not fully compatible with QoS/SQM" 提示。
- `luci-app-turboacc` 只存在于 `kenzok8/small-package/other/lean/`，其 Makefile 依赖 `kmod-ipt-offload`，init 脚本检测 `xt_FLOWOFFLOAD.ko` 并写 `firewall.@defaults[0].fullcone` —— 都是 iptables 时代的产物，与 25.12 的 nftables / firewall4 模块和键名不匹配。上一轮把它列为失效符号删掉是正确的。旧面板里的 DNS 缓存由 AdGuardHome + MosDNS 覆盖。

**本次新增**：

| 文件 | 作用 |
|---|---|
| `config.seed` 3 行 `kmod-*` | 把 BBR / 软件 Flow Offload / FullCone NAT 三个内核模块显式编入（兜底） |
| `files/etc/uci-defaults/99-acc-tuning` | 首次开机写入 `tcpcca=bbr`、`fullcone=1`、`fullcone6=1`、`flow_offloading=0`、`flow_offloading_hw=0`；执行完由系统自动删除 |
| `files/etc/sysctl.d/90-acc-tuning.conf` | `tcp_fastopen=3`、`tcp_mtu_probing=1`、`rmem_max/wmem_max=16M`。不改 `default_qdisc`（避免与 SQM 冲突），不覆盖 `tcp_rmem/tcp_wmem`（保留内核自动调优） |
| `.github/workflows/build.yml` | 新增「注入出厂加速默认值」步骤，把上述文件复制进 `lede/files/`（OpenWrt 会原样叠加到 rootfs），校验可执行位与关键行 |
| `apply.sh` | 复制这两个文件到仓库 |

**出厂默认组合**：BBR 开、FullCone NAT 开、软件 Flow Offload **关**。flowtable 会让已建立连接绕过 nftables 规则，与 SSR-Plus 透明代理 / SQM 冲突（典型症状「测速正常但分流失效」）。不用代理 / SQM 时在「网络 → 防火墙 → 区域设置」把 *Flow offloading type* 改成 `Software flow offloading` 即可，不必重编译也不必装包。x86/64 没有 HNAT / SFE / QCA-NSS-ECM，硬件卸载保持关闭。

## 六、工作流 / 配置改动

| 文件 | 改动 |
|---|---|
| `.github/workflows/build.yml` | 删除 `feed_mode` 输入与 `FEED_MODE` 环境变量；重写「配置软件源」步骤（仅官方 feed + helloworld 提到首行）；新增「注入出厂加速默认值」步骤（复制到 `lede/files/` + 落地校验）；`timeout-minutes` 330 → **345**；其余缓存、`-j2`、单次编译、产物校验、失败日志上传保持不变 |
| `config.seed` | 全部改为生效项（不再有「可选注释区」），按用途分组并写明每行理由；新增 ssr-plus 选项、底层服务包与 3 个 ACC 内核模块 |
| `files/etc/uci-defaults/99-acc-tuning` | **新增**：出厂 BBR / FullCone NAT 默认值，软件 Flow Offload 默认关 |
| `files/etc/sysctl.d/90-acc-tuning.conf` | **新增**：TCP Fast Open、MTU Probing、收发缓冲区上限 |
| `README.md` | 「两种 feed 组合」改为「单一官方 feed 来源」；更新触发方式、插件说明、风险提示 |
| `PLUGINS.md` | 从「推荐清单」改为「最终清单与取舍说明」 |
| `CHANGES.md` | 本文件 |
| `apply.sh` | 复制列表加入 `CHANGES.md` 与 `files/` 下两个加速配置文件；推送改用当前分支（不再硬编码 `main`）；未配置全局 git 身份时沿用仓库原提交作者（只写本仓库 config）；提交信息更新 |
| `openwrtBuild.patch` | 重新生成，包含全部新文件 |

## 七、验证状态与风险

**已验证**（本地）：YAML 可解析（**17 步**、timeout 345、仅 `lede_ref` 一个输入）；**9 段 `run` 脚本**与 `apply.sh`、`99-acc-tuning` 的 `bash -n` 全通过；`config.seed` 共 **51 个生效符号、0 重复、0 行内注释**；对照 `coolsnowwolf/luci@openwrt-25.12`（14205 项）、`coolsnowwolf/packages`（7935 项）、`fw876/helloworld`（1121 项）、`coolsnowwolf/lede`（26851 项）四棵完整 git 树逐一核对 **42 个真实包名（39 个 feed 包 + 3 个内核模块）全部存在，0 缺失**；三个 kmod 的 `KernelPackage` 定义也逐个读过 Makefile；补丁 `git apply --check` 与实际应用均通过，**8 files changed, 650 insertions, 175 deletions**；`apply.sh` 在模拟仓库提交成功（同样的 8 / 650 / 175）；补丁应用后的 7 个文件与源文件归一化逐字节一致，`99-acc-tuning` 在补丁中的 mode 为 `100755`。

> Windows 上 `git apply` 会把工作区文件转成 CRLF（提交时 git 会归一化回 LF），因此**推荐用 `apply.sh` 直接复制**（保持 LF），避免 `.config` 带 `\r`。

**每个新增包的 `LUCI_DEPENDS` 也已逐一读取**，确认没有依赖指向 kenzok8 独有包；`cpufreq` 因底层包缺失被排除，`dockerman` / `upnp` 的「界面不带服务」问题已显式补包处理。

**未验证**：没有在真实 GitHub runner 上跑过完整编译（GitHub 连接器写权限仍为只读，见下）。

风险与退路：

1. **首次编译时间**：33 个 LuCI 应用里有十几个 Go 程序（xray、mihomo、sing-box、tailscale、lucky、openlist、syncthing、adguardhome、mosdns…），首次全量编译可能 4–6 小时。`dl/` 与 `.ccache` 缓存对第二次起效，所以定时编译已改每周一次。
2. **Docker 是唯一明显拉长编译的项**（要编多个内核模块）。若首次构建因它超时/失败，注释掉 `config.seed` 里 `dockerman` / `dockerd` / `docker` 三行即可，其余不受影响。
3. **选项类符号可能出现在 warning**：`INCLUDE_*` 是 Kconfig `choice`，工作流的「显式包未生效数」偶尔会列出它们，不影响编译；`xray-core`、`mihomo` 已单独写进 seed 作为兜底。
4. **ADG 与 MosDNS 需手动串联**：固件编出来后要在页面里定好谁监听 53、谁做上游，否则可能解析循环。
5. **软件 Flow Offload 出厂默认关闭**，与 SSR-Plus 透明代理 / SQM 互斥，强开会出现「测速正常但分流失效」；不用代理 / SQM 时在防火墙页面自行打开即可。x86 无硬件 NAT，硬件卸载不要开。
6. 产物仍是 **ipk**（Lean 主线用 opkg）；`openwrt-25.12` 只是 LuCI feed 分支名。
