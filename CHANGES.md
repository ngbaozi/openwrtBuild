# 本次改动列表（2026-11-11）

**目标**：所有插件一次编进固件，后续不再 opkg 安装；消除同名包重复；来源收敛到官方 feed，保证可编译。

**结果**：编译包从 **13 个 CONFIG_PACKAGE 符号 → 53 个**（其中 LuCI 应用/主题 33 个、底层服务 8 个、ACC 加速内核模块 3 个、ssr-plus 选项 9 个）；第三方 feed 从 2 个 → 0 个；重复代理面板从 2 个 → 1 个；官方 feed 之间的同名包已逐项裁决（详见第十一节）。

> **20:18 追加**：去广告由 AdGuardHome 换成官方 `adblock-fast`，并注入出厂启用脚本。详见第六节；这是在同一次交付里的追加改动，前面各节的统计已同步更新。改动已推送到 `ngbaozi/openwrtBuild` 的 `main` 分支。
>
> **11-12 08:46 追加**：首次真实云编译（11-11 run #1）**超时失败**。已按实际运行数据修复并行度与缓存策略，详见第九节；本文档第七、八节的统计已同步更新。
>
> **11-12 14:43 追加**：run #2 编译成功（4h37m），但刷机后 LuCI 显示「MosDNS 未运行」。根因是 `helloworld` 与 `packages` 的同名 `mosdns` 包版本错配，已定向修复并新增出厂启用脚本，详见第十一节。符号 52 → **53**。

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

> 可行性核实：拉取 `coolsnowwolf/packages` 完整 git 树（7935 项）逐一比对，官方 feed 已含 `adblock-fast`、`ddns-go`、`lucky`、`openlist`、`dockerd`、`docker`、`ttyd`、`watchcat`、`nlbwmon`、`banip`、`sqm-scripts`、`irqbalance`、`netdata`、`geo2txt`、`v2ray-geodata`、`sing-box`、`xray-core`、`tailscale`、`zerotier`、`qBittorrent`、`transmission`、`syncthing`、`msd_lite`、`udpxy`、`UnblockNeteaseMusic`、`uugamebooster`、`filebrowser`、`mwan3` 等全部底层依赖。

## 二、删除的插件 / 符号

| 删除项 | 原因 |
|---|---|
| `luci-app-passwall` | 与 `ssr-plus` 同为透明代理面板，同时启用会争抢 nftables 规则与 53 端口；协议三家都全，留官方同源的 ssr-plus |
| `haproxy` | 原为 PassWall 负载均衡依赖，面板移除后无使用者 |
| `luci-app-adguardhome`（20:18 追加） | Go 常驻进程 + 独立监听 53，与 MosDNS 需手工串联才不解析循环；改用 `adblock-fast` |
| 6 个失效符号（`adbyby-plus`、`turboacc`、`turboacc-config`、`ap-modem`、`broadbandacc`、`fastnet`） | 上一轮已删，feed 中不存在，defconfig 会静默丢弃 |

**DNS 组合现在更简单**：`adblock-fast` 不监听端口，只在 dnsmasq 层注入黑名单；MosDNS 继续做国内外分流，不再需要「谁监听 53」的串联配置，也不再有解析循环风险。

## 三、新增插件（24 个 LuCI 应用/主题 + 5 个底层服务 + 3 个 ACC 内核模块 + 9 个 ssr-plus 选项）

| 分组 | 新增项 | 作用 |
|---|---|---|
| **基础运维（10）** | `filetransfer`、`ttyd`、`diskman`、`nlbwmon`、`watchcat`、`oui`、`sysctl`、`arpbind`、`wol`、`timecontrol` | 网页装包 / 网页终端 / 磁盘管理 / 客户端流量 / 断网自愈 / 厂商图标 / 内核参数 / IP-MAC 绑定 / 网络唤醒 / 时段控制 |
| **远程访问（2）** | `tailscale-community`、`lucky` | 异地组网；DDNS + 端口转发 + 反向代理 + 证书。因 lucky 已含 DDNS，**不再加** `ddns-go`、`frpc`、`natmap`、`zerotier` |
| **网络质量（5）** | `sqm`、`upnp`、`banip`、`irqbalance`、`attendedsysupgrade` | 抗 bufferbloat / UPnP（自动拉 `miniupnpd-nftables`）/ 恶意 IP 封禁 / 多核中断均衡 / 在线升级 |
| **重量级（4）** | `dockerman`、`openlist`、`qbittorrent`、`syncthing` | Docker 管理 / 多网盘挂载 / BT 下载 / 文件同步。加 `transmission` 会与 qBittorrent 重复，故不加 |
| **主题（3）** | `luci-theme-design`、`argon-config`、`design-config` | Design 主题（11-08 起上游默认）+ 两套主题的设置页；两种主题风格不同，运行时切换 |
| **底层服务（5）** | `xray-core`、`mihomo`、`dockerd`、`docker`、`adblock-fast` | `dockerman` 的 Makefile 不带 `+dockerd`，必须显式补服务与 CLI；xray/mihomo 兜底确保内核编入；`adblock-fast` 显式写 y 保证去广告服务一定在固件里 |
| **ssr-plus 选项（9）** | `Nftables_Transparent_Proxy`、`INCLUDE_Xray`、`Shadowsocks_Rust_Client`、`Mihomo`、`GeoData`、`ChinaDNS_NG`、`MosDNS`、`Shadow_TLS`、`Http_Proxy` | 默认 `V2ray-core Selection = None`，**不显式开 `INCLUDE_Xray` 就没有 Reality / VLESS**；其余把常用协议与 DNS 组件一次开齐 |
| **ACC 内核模块（3）** | `kmod-tcp-bbr`、`kmod-nft-offload`、`kmod-nft-fullcone` | BBR / 软件 Flow Offload / nftables FullCone NAT。官方 `firewall4` 的 Makefile 已把它们列为 `DEPENDS`，显式写 `=y` 是兜底，确保一定编入 |

## 四、保留的插件（8 个 LuCI + 2 个底层）

`ssr-plus`、`mosdns`、`filebrowser`、`mwan3`、`syncdial`、`uugamebooster`、`unblockneteasemusic`、`luci-theme-argon`；底层 `v2ray-core`、`sing-box`。（`adguardhome` 已按第六节移除。）

## 五、网络加速（ACC）专项

**结论：不需要 `luci-app-turboacc`，官方 feed 里本来就有同等能力。** 核实过程：

- `coolsnowwolf/lede@master` 的 `package/network/config/firewall4/Makefile` 中，`Package/firewall4` 的 `DEPENDS` 已含 `+kmod-tcp-bbr +kmod-nft-offload +kmod-nft-fullcone`。其中 `kmod-tcp-bbr` 定义在 `package/kernel/linux/modules/netsupport.mk` 并自带 `/etc/sysctl.d/12-tcp-bbr.conf`（写入 `net.ipv4.tcp_congestion_control=bbr`），`kmod-nft-offload` 在 `netfilter.mk`，`kmod-nft-fullcone` 在 `package/network/services/fullconenat-nft/Makefile`。
- `coolsnowwolf/luci@openwrt-25.12` 的 `luci-app-firewall/.../view/firewall/zones.js` 已内置 `fullcone`、`fullcone6`、`tcpcca` 选项以及 `Flow offloading type`（None / Software / Hardware / MTK HNAT / QCA ECM），并自带 "Not fully compatible with QoS/SQM" 提示。
- `luci-app-turboacc` 只存在于 `kenzok8/small-package/other/lean/`，其 Makefile 依赖 `kmod-ipt-offload`，init 脚本检测 `xt_FLOWOFFLOAD.ko` 并写 `firewall.@defaults[0].fullcone` —— 都是 iptables 时代的产物，与 25.12 的 nftables / firewall4 模块和键名不匹配。上一轮把它列为失效符号删掉是正确的。旧面板里的 DNS 缓存一项，本固件由 dnsmasq + MosDNS 承担。

**本次新增**：

| 文件 | 作用 |
|---|---|
| `config.seed` 3 行 `kmod-*` | 把 BBR / 软件 Flow Offload / FullCone NAT 三个内核模块显式编入（兜底） |
| `files/etc/uci-defaults/99-acc-tuning` | 首次开机写入 `tcpcca=bbr`、`fullcone=1`、`fullcone6=1`、`flow_offloading=0`、`flow_offloading_hw=0`；执行完由系统自动删除 |
| `files/etc/sysctl.d/90-acc-tuning.conf` | `tcp_fastopen=3`、`tcp_mtu_probing=1`、`rmem_max/wmem_max=16M`。不改 `default_qdisc`（避免与 SQM 冲突），不覆盖 `tcp_rmem/tcp_wmem`（保留内核自动调优） |
| `.github/workflows/build.yml` | 新增「注入出厂默认值」步骤，把上述文件复制进 `lede/files/`（OpenWrt 会原样叠加到 rootfs），校验可执行位与关键行 |
| `apply.sh` | 复制这些文件到仓库 |

**出厂默认组合**：BBR 开、FullCone NAT 开、软件 Flow Offload **关**。flowtable 会让已建立连接绕过 nftables 规则，与 SSR-Plus 透明代理 / SQM 冲突（典型症状「测速正常但分流失效」）。不用代理 / SQM 时在「网络 → 防火墙 → 区域设置」把 *Flow offloading type* 改成 `Software flow offloading` 即可，不必重编译也不必装包。x86/64 没有 HNAT / SFE / QCA-NSS-ECM，硬件卸载保持关闭。

## 六、去广告方案替换（20:18 追加）

**需求**：不要 AdGuardHome，换另一个去广告方案，仍然要求刷完即用、后续不装包。

**核实**（五棵 feed 树 + Makefile 逐一读）：官方 feed 里可用的 DNS 去广告包有三个 —— `net/adblock-fast` + `applications/luci-app-adblock-fast`、`net/adblock` + `applications/luci-app-adblock`、`net/simple-adblock`（无 LuCI 面板）。`luci-app-adbyby-plus` 只在 `kenzok8/small-package`，且是 iptables 时代的 HTTP 广告过滤，第三方 feed 已全部移除，不可选。

**选 `adblock-fast` 的理由**：

| | AdGuardHome（删） | adblock-fast（用） | 经典 adblock（未选） |
|---|---|---|---|
| 形态 | Go 二进制常驻、独立监听 53 | ucode 脚本，生成黑名单后退出 | shell 脚本 |
| LuCI 依赖 | `+adguardhome +luci-base` | `+luci-base +adblock-fast +rpcd-mod-ucode` | 依赖 `firewall4 +coreutils +gawk +rpcd` 等，更重 |
| 与 MosDNS | 需手工串联，否则解析循环 | 写进 dnsmasq（`dnsmasq.servers`），不占端口，天然不冲突 | 同为 dnsmasq 后端，但依赖更多 |
| 面板 | 有 | 有（服务 → AdBlock-Fast） | 有 |

**关键坑**：`net/adblock-fast/files/etc/config/adblock-fast` 里 `enabled '0'`，且 17 个内置规则源全部 `enabled '0'`。只把包编进固件 = 装了个空面板，什么都不拦。

| 文件 | 改动 |
|---|---|
| `config.seed` | 删除 `CONFIG_PACKAGE_luci-app-adguardhome=y`；新增 `luci-app-adblock-fast=y` 与 `adblock-fast=y`（显式兜底）；DNS 段注释与「有意不编入」清单同步改写。符号 51 → **52** |
| `files/etc/uci-defaults/98-adblock-fast` | **新增**（mode 100755）：首次开机 `enabled='1'`、后端 `dnsmasq.servers`、启用 Hagezi Pro 与 OISD Big 两个不同 CDN 的规则源、`/etc/init.d/adblock-fast enable`；执行完系统自动删除。包名前缀 98 保证在包自带的 `90-adblock-fast` 之后运行 |
| `.github/workflows/build.yml` | 「注入出厂加速默认值（ACC）」改为「注入出厂默认值（ACC + 去广告）」：多复制 98 脚本、校验可执行位与 `adblock-fast.config.enabled='1'` 一行 |
| `apply.sh` | 复制列表加入 98 脚本并 `chmod +x`；提交信息更新 |
| `README.md` / `PLUGINS.md` / `CHANGES.md` | 新增方案对比、删掉「ADG 与 MosDNS 需手动串联」的旧提示、更新统计 |
| `openwrtBuild.patch` | 重新生成，包含全部新文件 |

**出厂效果**：刷机后 adblock-fast 已启用、开机自启、规则源已选好，WAN 就绪后自带 procd 触发器自动重试下载。换源 / 白名单 / 暂停都在「服务 → AdBlock-Fast」，不需要命令行、不需要装包。想换回 AdGuardHome 只需把 seed 里两行改回并删掉 98 脚本。

## 七、工作流 / 配置改动

| 文件 | 改动 |
|---|---|
| `.github/workflows/build.yml` | 删除 `feed_mode` 输入与 `FEED_MODE` 环境变量；重写「配置软件源」步骤（仅官方 feed + helloworld 提到首行）；新增「注入出厂默认值」步骤（复制 3 个文件 + 落地校验）；`timeout-minutes` 330 → 345 → **355**；并行度 `-j2` → **自适应 -j8**；缓存改 `restore` + `save` 分离；编译命令加软超时（详见第九节）；产物校验、失败日志上传保持不变 |
| `config.seed` | 全部改为生效项（不再有「可选注释区」），按用途分组并写明每行理由；新增 ssr-plus 选项、底层服务包、3 个 ACC 内核模块；去广告两行替换 |
| `files/etc/uci-defaults/99-acc-tuning` | **新增**：出厂 BBR / FullCone NAT 默认值，软件 Flow Offload 默认关 |
| `files/etc/uci-defaults/98-adblock-fast` | **新增**：出厂启用 adblock-fast 并预置规则源 |
| `files/etc/sysctl.d/90-acc-tuning.conf` | **新增**：TCP Fast Open、MTU Probing、收发缓冲区上限 |
| `README.md` | 「两种 feed 组合」改为「单一官方 feed 来源」；新增去广告小节；更新文件表、风险提示与落地说明 |
| `PLUGINS.md` | 从「推荐清单」改为「最终清单与取舍说明」；去广告行与去重表同步 |
| `CHANGES.md` | 本文件 |
| `apply.sh` | 复制列表加入 `CHANGES.md` 与 `files/` 下三个出厂配置文件；推送改用当前分支（不再硬编码 `main`）；未配置全局 git 身份时沿用仓库原提交作者（只写本仓库 config）；**`git update-index --chmod=+x` 显式写 mode** —— Windows 上 `core.filemode=false`，只做 `chmod +x` 提交出来仍是 `100644`，首次开机脚本可能不执行；提交信息更新 |
| `openwrtBuild.patch` | 重新生成，包含全部新文件（相对最初仓库的完整补丁） |

## 八、验证状态与风险

**已验证**（本地）：YAML 可解析（**19 步**、timeout 355、仅 `lede_ref` 一个输入）；**10 段 `run` 脚本**与 `apply.sh`、两个 uci-defaults 脚本的 `bash -n` 全通过；`actions/cache/restore@v6` / `save@v6` 的 `action.yml` 与 `cache-hit` 输出已核对；`config.seed` 共 **52 个生效符号、0 重复、0 行内注释**；对照 `coolsnowwolf/luci@openwrt-25.12`（14205 项）、`coolsnowwolf/packages`（7935 项）、`fw876/helloworld`（1121 项）、`coolsnowwolf/lede`（26851 项）四棵完整 git 树逐一核对 **43 个真实包名（40 个 feed 包 + 3 个内核模块）全部存在，0 缺失**；`adblock-fast` 与 `luci-app-adblock-fast` 的 `LUCI_DEPENDS` / `DEPENDS`、默认 UCI 配置、uci-defaults、init.d 均已读过；补丁 `git apply --check` 与实际应用通过；`apply.sh` 在模拟仓库提交成功；补丁应用后的全部文件与源文件归一化逐字节一致，两个 uci-defaults 脚本在补丁中 mode 为 `100755`。

> Windows 上 `git apply` 会把工作区文件转成 CRLF（提交时 git 会归一化回 LF），因此**推荐用 `apply.sh` 直接复制**（保持 LF），避免 `.config` 带 `\r`。

**每个新增包的 `LUCI_DEPENDS` 也已逐一读取**，确认没有依赖指向 kenzok8 独有包；`cpufreq` 因底层包缺失被排除，`dockerman` / `upnp` 的「界面不带服务」问题已显式补包处理。

**已推送**：`main` 分支，改动以本地文件 + `openwrtBuild.patch` 同步归档。

**真实 runner 结果**：11-11 手动 run #1 **失败（超时取消）**，详见第九节；11-12 的修复尚未在 runner 上复跑。**未验证**：真机上的广告拦截效果与出厂 UCI 默认值。

风险与退路：

1. **首次编译时间**：33 个 LuCI 应用里有十几个 Go 程序（xray、mihomo、sing-box、tailscale、lucky、openlist、syncthing、mosdns…；换掉 AdGuardHome 后少一个），首次全量编译约 3–5 小时。旧配置 `-j2` 实测 5h21m 仍未完成（第九节），改 `-j8` 后预期显著缩短；`dl/` 与 `.ccache` 现在即使失败也会保存，第二次起效。
2. **Docker 是唯一明显拉长编译的项**（要编多个内核模块）。若修复后仍超时/失败，注释掉 `config.seed` 里 `dockerman` / `dockerd` / `docker` 三行即可，其余不受影响。
3. **选项类符号可能出现在 warning**：`INCLUDE_*` 是 Kconfig `choice`，工作流的「显式包未生效数」偶尔会列出它们，不影响编译；`xray-core`、`mihomo` 已单独写进 seed 作为兜底。
4. **规则源需联网**：adblock-fast 的黑名单是开机后下载的，出厂预置两个不同 CDN 的规则源；若国内都拉不通，到「服务 → AdBlock-Fast」换一个可访问的域名列表即可，不用重编译。
5. **软件 Flow Offload 出厂默认关闭**，与 SSR-Plus 透明代理 / SQM 互斥，强开会出现「测速正常但分流失效」；不用代理 / SQM 时在防火墙页面自行打开即可。x86 无硬件 NAT，硬件卸载不要开。
6. 产物仍是 **ipk**（Lean 主线用 opkg）；`openwrt-25.12` 只是 LuCI feed 分支名。
7. 工作流**只有定时与手动触发，没有 push 触发**；改完配置要自己去 Actions 里 Run workflow。

---

## 九、首次云编译超时排障与修复（11-12 追加）

### 实测数据（公开 API 读取 run #1 / `43020658088`）

| 项 | 值 |
|---|---|
| 结论 | `cancelled`，annotation：`The job has exceeded the maximum execution time of 5h45m0s` |
| job 总耗时 | 5:45:40（`12:28:44Z → 18:14:24Z`） |
| 准备阶段 | 释放磁盘 2m41s、依赖 31s、feed 1m01s、defconfig 25s、**下载源码 19m31s**，合计约 24 分钟 |
| **编译固件** | **321m17s（5h21m）后 `cancelled`** |
| 后续步骤 | 产物整理、3 个上传、失败日志上传、**`Post 恢复 dl 与 ccache` 全部 `skipped`** |

对照历史：旧工作流 `bulid.yml`（22 个 `CONFIG_PACKAGE` 符号、`-j$(nproc*2)` = **-j8**）在 11-10 run #132 **成功，总耗时 4h57m**；11-08 / 11-09 两次也因平台 6h 硬上限被取消。

### 三个根因

1. **并行度写错**：工作流按「标准 runner 仅 2 核 / 7GB」把 `nproc=4` 强制压到 `-j2`。查 GitHub 官方 runner 规格表：**公开仓库** `ubuntu-22.04` = **4 vCPU / 16GB / 14GB SSD**（2 核 / 7GB 是旧版私有仓库规格）。一半算力被闲置；包数还从 22 涨到 52。
2. **缓存从未落地**：`actions/cache@v6` 一体式用法靠 **Post 步骤**保存缓存，而 job 被超时取消时 Post 直接 `skipped` —— 本次 `dl/` 与 `.ccache` 全丢，下次仍从零编。
3. **硬取消让所有收尾失效**：`timeout-minutes: 345` 触发后，失败日志上传（`if: failure()`）也被跳过，`build.log` 拿不到，无法定位卡在哪个包。

### 修复（只动工作流，插件清单不变）

| 改动 | 前 | 后 |
|---|---|---|
| 并行度 | `JOBS=$(( CORES > 2 ? 2 : CORES ))` → `-j2` | 读 `/proc/meminfo`：≥12GB 用 `CORES*2`，否则 `CORES`，上限 8 → 实测 **`-j8`** |
| job 上限 | `timeout-minutes: 345` | **355**（平台硬上限 360） |
| 编译软超时 | 无 | 第一步写 `BUILD_START`；编译步骤算 `DEADLINE = BUILD_START + 320min`，用 `timeout --signal=INT --kill-after=2m "${LEFT}s" make` 主动收尾，留 **35 分钟**给缓存保存与日志上传 |
| 缓存 | `actions/cache@v6`（Post 保存） | `actions/cache/restore@v6`（`id: cache-restore`）+ 紧跟编译、`if: always()` 的 `actions/cache/save@v6`；精确命中本周 key 时不重复保存 |
| ccache 上限 | 2G | **4G**（`rules.mk` 已确认默认 `CCACHE_DIR=$(TOPDIR)/.ccache`，路径 `lede/.ccache` 正确） |
| 下载 | `make download -j8`（19m31s） | `-j16`，失败仍回退 `-j1 V=s` |
| 失败可观测 | `build.log` 因 job 取消拿不到 | 软超时后 job 是 `failure` 而非 `cancelled`，`if: failure()` 的日志上传必跑 |

### 预期与退路

- `-j8` 相对 `-j2` 理论提速约 2–3×；同仓库旧配置 `-j8` 在更少包的情况下 4h57m 成功。首次全量预计 3–5 小时，落在 355 分钟内。
- 即使首次仍触发软超时，本次的 `dl/` 与 `.ccache` 会被显式保存，**第二次运行命中 ccache 后续编**，通常能完成；失败时 `build.log` 会作为 artifact 上传，可直接定位。
- 若连续两次都不行，按风险第 2 条注释掉 Docker 三行再跑；或把 `config.seed` 里 `openlist` / `qbittorrent` / `syncthing` / `tailscale` 等 Go 包暂时下线。

**复跑状态（11-12 09:05）**：run #2（`43131193707`）已于 UTC 00:58 启动，`生成并校验 .config`、`下载源码包` 均 success，已进入 `编译固件` 步骤；check-run annotations **0 条**，即 52 个符号全部生效、没有「配置项未生效」告警。编译结果待定。

---

## 十、`defconfig` 里成片「依赖不存在」警告的核实（11-12 追加）

**结论：无害，不是编译错误，与本固件选中的包无关。** 这是上游两个 feed 之间版本不匹配的元数据警告，OpenWrt 打印后忽略该依赖继续生成 `.config`。

### 完整来源链

| 环节 | 事实 |
|---|---|
| `feeds.conf.default` | 实际有 **6 个**官方 feed：`packages` / `luci` / `routing` / `telephony` / `helloworld` / **`video`**。此前文档写成「五个」，本次已修正 |
| `video/libs/libdrm/Makefile` | 定义了 `libdrm-amdgpu` / `-radeon` / `-intel` / `-tegra` / `-etnaviv` / `-nouveau` 等子包 |
| `packages/libs/libdrm/Makefile` | **只定义单个 `libdrm` 包**（`$(eval $(call BuildPackage,libdrm))`），没有上述子包 |
| `feeds install -a` | 按 `feeds.conf` 顺序取先出现的同名包 → `packages` 在 `video` 前，**选中 packages 版 libdrm**，video 版子包全部「不存在」 |
| `video/libs/mesa/Makefile` | `DEPENDS+=... +libdrm-amdgpu +libdrm-radeon ...`，逐项触发警告 |
| `video/utils/kmscon/Makefile` | `DEPENDS:=input-support ...`，而 `input-support` 在 video 与 packages 两棵完整 git 树里**都没有定义** |
| 重复几十行 | `make defconfig` 多轮扫描同一 Makefile 反复打印，不是几十个独立故障 |

### 为什么确定不影响本固件

- `config.seed` 的 **43 个真实包名**与 video feed 的 57 个包目录做交集 = **0 个**；也没有任何 `mesa` / `kmscon` / `libdrm` / `qt5` / `wayland` / `vulkan` 相关符号。
- 这些警告由 `defconfig` 扫描**全部** feed Makefile 产生，与被选中与否无关；x86/64 软路由固件不编图形栈。
- run #2 的 `生成并校验 .config` 步骤已 **success**，check-run annotations **0 条** —— 工作流里 `::warning::配置项未生效` 一条都没触发，52 个符号全部生效。
- 截图末尾 `make[1] tools/flock/compile` 说明构建已正常进入编译阶段。

### 本次改动（只加解释，不动 feed 结构）

| 文件 | 改动 |
|---|---|
| `.github/workflows/build.yml` | 「配置软件源」注释改为 6 个 feed 并说明 video 来源；`make defconfig` 改为 `2>&1 \| tee defconfig.log`；Step Summary 新增「上游 video feed 元数据警告 N 条」归类统计；编译步骤开头新增一条 `::notice::` 说明这些警告无害 |
| `README.md` | feed 数量五个→六个；「编译前请留意」新增一条完整解释 |
| `CHANGES.md` | 本节 |

> [!WARNING] 为什么暂不删 video feed
> 注释掉 `src-git video` 或删 `package/feeds/video/{mesa,kmscon}` 符号链接确实能让日志干净，但会改变依赖解析环境。run #2 正在验证 `-j8` 修复，**此时改 feed 结构会让失败原因无法归因**。等这次编译成功后，若要彻底静音再单独处理，风险更低。

**真正该盯的三个信号**：① 「显式包未生效数」是否为 0；② 编译步骤是否报错；③ 产物步骤是否生成固件。这些都不受上述警告影响。

## 十一、刷机后「MosDNS 未运行」：同名包版本错配（11-12 追加）

**背景**：run #2（`43131193707`）在 11-12 05:35 UTC **success**，总耗时 4h37m（编译步骤 266 分钟），固件与 ipk 全部产出；`-j8` + `restore/save` 缓存修复得到验证。刷机后 LuCI「服务 → MosDNS」顶部显示红字 **MosDNS 未运行**（版本 `V5.3.4`，已启用 ✅、监听 5335、内置预设、DNS 转发 ✅、DNS 重定向 ⬜）。

### 结论：不是没装包，是 `mosdns` 二进制装错了版本

`feeds.conf.default` 把 `helloworld` 提到首行（为了 xray-core / mihomo 的 EIP93 加速补丁），而 **`helloworld` 与 `packages` 都定义了源包 `mosdns`**。`feeds install -a` 按 conf 顺序取先出现的同名包，于是装成了 helloworld 版。

**run #2 构建日志原文（决定性证据）**：

```
##[notice]首行 feed：src-git helloworld https://github.com/fw876/helloworld.git
Installing all packages from feed helloworld.
Installing package 'mosdns' from helloworld
Installing package 'luci-app-mosdns' from luci
 make[2] -C feeds/helloworld/mosdns download
```

| | `fw876/helloworld/mosdns`（**实际编入**） | `coolsnowwolf/packages/net/mosdns`（**应该用**） |
|---|---|---|
| PKG_VERSION / RELEASE | `5.3.4` / `1` | `5.3.4` / `15` |
| 补丁数 | **3**（203 / 204 / 205） | **30**（含 `216-plugin-add-stats_api-plugin.patch`、`207-add-adblock_set-database-plugin.patch`、日志轮转、cache prefetch、query log…） |
| 能加载 LuCI 生成的配置吗 | **不能** | 能 |

`luci-app-mosdns`（`luci` feed，1.7.15）的 `/etc/init.d/mosdns` 用 `jshn` 生成 `/var/etc/mosdns.json`，其中无条件带 `type: adblock_set`（blocklist 插件），`stats_collector=1` 时还带 `type: stats_api`。这两个都是 packages 版补丁新增的插件类型，helloworld 版二进制遇到就是 `unknown plugin` → 启动即退出。LuCI 的「未运行」来自 `callServiceList('mosdns')` 读 procd 实例的 `running` 状态，所以进程反复 respawn / 退出时页面就一直是红字，统计页也会失效（`luci.mosdns` 依赖 `127.0.0.1:$listen_port_api/plugins/stats_collector/...`，同样需要 `stats_api` 补丁）。

### 本次改动（10 个文件）

| 文件 | 改动 |
|---|---|
| `.github/workflows/build.yml` | 「配置软件源」步骤在 `feeds install -a` 之后执行 `./scripts/feeds uninstall mosdns` + `./scripts/feeds install -p packages mosdns`，再用 `find` + `readlink -f` **强制校验**软链接唯一且指向 `$PWD/feeds/packages/net/mosdns`，否则 `::error::` 直接失败。其余同名包（xray-core / mihomo）保持 helloworld 加速版。步骤名由「零同名包冲突」改为「官方 feed + 同名包定向裁决」 |
| `config.seed` | DNS 段新增 `CONFIG_PACKAGE_mosdns=y` 显式兜底（符号 **52 → 53**，底层服务 7 → 8）；注释写明版本要求与裁决方式；首行「零第三方重复包」改为「同名包已定向裁决」 |
| `files/etc/uci-defaults/97-mosdns` | **新增**（100755）：官方 `/etc/config/mosdns` 默认 `enabled='0'`，首次开机置 1、确认 `configfile=/var/etc/mosdns.json`、`redirect=1`、`local_dns_redirect=0`，并 `/etc/init.d/mosdns enable`。刷完不用手动勾「已启用」 |
| `.github/workflows/build.yml` | 「注入出厂默认值」改名（ACC + 去广告 + **DNS**），多复制并校验 97 脚本 |
| `apply.sh` | 复制 / chmod / `git update-index --chmod=+x` 三个 uci-defaults 脚本；提交信息更新 |
| `README.md` / `PLUGINS.md` / `CHANGES.md` | 新增「DNS 分流」小节、排查顺序、同名包裁决说明；修正 `PLUGINS.md` 里「不存在同名包被覆盖的问题」的过时结论与 `timeout-minutes: 345` 旧值 |
| `openwrtBuild.patch` | 重新生成，含 97-mosdns |

**没有改 feed 顺序**，也没有删 helloworld 的加速补丁 —— 只针对 mosdns 这一个包做反向裁决，把副作用降到最小。`luci-app-ssr-plus` 自带的 `mosdns-config.yaml` 只用 `cache` / `forward` / `sequence` / `udp_server` / `tcp_server` 标准插件，官方版二进制完全兼容。

### 当前这台设备怎么救急（不用等重新编译）

> [!DANGER] 「DNS 转发」已开而服务没起来会断网
> dnsmasq 的上游被指向 `127.0.0.1#5335`，端口没人监听就全网卡在解析。先恢复上网：
> ```sh
> uci set mosdns.config.redirect=0; uci commit mosdns
> /etc/init.d/mosdns stop; /etc/init.d/dnsmasq restart
> ```

之后按需要二选一：

1. **推荐：等重新编译刷机**（修复后的固件里 mosdns 是 `5.3.4-15`，且出厂已启用）。Actions → Run workflow，约 4.5 小时，产物在 Artifacts。
2. **临时让现有固件跑起来**（权宜之计，统计页与内置 blocklist 会失效；去广告本来由 adblock-fast 在 dnsmasq 层承担，不受影响）：把生成配置里两个不兼容插件换掉再重启 ——
   ```sh
   sed -i 's/json_add_string "type" "adblock_set"/json_add_string "type" "domain_set"/' /etc/init.d/mosdns
   uci set mosdns.config.stats_collector=0; uci commit mosdns
   /etc/init.d/mosdns restart
   cat /var/log/mosdns.log     # 若仍有 unknown plugin，说明还有补丁专属字段，需换自定义 yaml
   ```
   这不是根治：helloworld 版还缺日志轮转、cache prefetch 等补丁，LuCI 的高级选项可能对不上。

排查命令速查：`/etc/init.d/mosdns status`、`logread | grep -i mosdns`、`opkg info mosdns | grep Version`（应为 `5.3.4-15` 而非 `-1`）。远程 DNS 默认 `tls://8.8.8.8`，国内 853 常被干扰；境外域名解析超时时换成可达的加密 DNS 或指向 SSR Plus+ 的本地 DNS 端口 —— 这项与网络环境相关，不写死在固件里。

### 验证状态

本地已完成：YAML 可解析（19 步、timeout 355）、10 段 `run` 与 4 个脚本 `bash -n`；`config.seed` 53 个符号 0 重复 0 行内注释；补丁 `git apply --check` 与应用逐字节一致；`apply.sh` 模拟提交检查三个 uci-defaults 的 `100755`。

**尚未验证**：修复后的固件还没编译、没刷机，`feeds uninstall` + `install -p packages` 只在源码逻辑与 `scripts/feeds` 实现层面核对过（`install_src` 的 `%installed` 机制决定必须先 uninstall 才能覆盖），需要在 runner 上复跑确认软链接校验通过。
