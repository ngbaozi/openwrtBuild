# openwrtBuild

在 GitHub Actions 上云编译 `coolsnowwolf/lede`（Lean OpenWrt）**x86/64** 固件。所有插件一次编进固件，刷完即可用，不需要后续 `opkg install`。

## 文件

| 文件 | 作用 |
|---|---|
| `config.seed` | 唯一的编译配置来源：目标平台、ccache、全部要编入的插件与 ssr-plus 选项 |
| `CHANGES.md` | 本次改动的完整列表（来源、增、删、留、工作流、去广告替换） |
| `PLUGINS.md` | 最终插件清单、去重逻辑与使用注意 |
| `files/etc/uci-defaults/99-acc-tuning` | 出厂网络加速默认值（BBR / FullCone NAT 开，软件 Flow Offload 关） |
| `files/etc/uci-defaults/98-adblock-fast` | 首次开机启用 adblock-fast 去广告并预置规则源 |
| `files/etc/sysctl.d/90-acc-tuning.conf` | TCP Fast Open / MTU Probing / 收发缓冲区 |
| `.github/workflows/build.yml` | 工作流（原 `bulid.yml` 拼写错误，已删除） |

## 触发方式

- **定时**：每周一 UTC 02:00（北京时间 10:00）。cron 恒按 UTC，与 runner 时区无关。
- **手动**：Actions → Lean OpenWrt x86/64 云编译 → Run workflow，可填 `lede_ref`（分支名或 commit SHA）。填 SHA 可复现某次构建 —— 上游 `master` 正在迁移 6.18 内核，不 pin 时今天能编明天不一定。
- **没有 `push` 触发**：改完配置后要手动 Run workflow 才出固件。

## 插件来源：只用官方 feed

工作流不再引入任何第三方 feed，只用 lede 自带的 `packages` / `luci` / `routing` / `telephony` / `helloworld` 五个官方源，并把 `helloworld` 提到 `feeds.conf.default` **首行**。

- 原因一：`kenzok8/openwrt-packages` 与 `kenzok8/small` 和官方 feed 有十几个同名包（`adguardhome`、`filebrowser`、`ddns-go`、`lucky`、`mosdns`、`ssr-plus`、`passwall`、`xray-core`…），OpenWrt 的 `feeds install` 按 conf 顺序取先出现的版本，结果不可控。
- 原因二：官方 `coolsnowwolf/packages` 已包含全部需要的底层依赖，且 golang 已是 **1.27.3**（比 kenzok8 README 要求的 1.26 更新），不需要再替换 golang 或删除重复包。
- 原因三：`helloworld` 首行后，选中的是它带 **EIP93 硬件 AES 加速补丁**与 AnyTLS / TUIC / Snell / ShadowTLS / Kcptun 协议补丁的 `xray-core` / `mihomo`，而不是官方 packages 的上游同名版本。

## 去广告：出厂已带（adblock-fast，不用 AdGuardHome）

去广告改用官方 `coolsnowwolf/packages@master/net/adblock-fast` + `coolsnowwolf/luci@openwrt-25.12/applications/luci-app-adblock-fast`，在 `config.seed` 里两个符号都显式写 `=y`。

| 对比 | AdGuardHome（已删） | adblock-fast（现用） |
|---|---|---|
| 形态 | Go 二进制常驻，独立监听 53 | ucode 脚本，生成黑名单后退出，不监听端口 |
| 与 MosDNS | 需手工串联、只让一个进程占 53，否则解析循环 | 黑名单写进 dnsmasq（`dnsmasq.servers`），与 MosDNS 分流天然不冲突 |
| 内存 / 编译 | 几十 MB 常驻 + 一个 Go 编译目标 | 几乎不占内存，纯脚本包 |
| 依赖 | `+adguardhome` | `+curl +resolveip +ucode* +gawk/grep/sed/sort` |

**必须注入出厂默认值**：官方包自带的 `/etc/config/adblock-fast` 里 `enabled='0'`、所有规则源也是 `enabled='0'`，只编进包等于装了个空面板。`files/etc/uci-defaults/98-adblock-fast` 在首次开机把服务置 1、启用 Hagezi Pro 与 OISD Big 两个不同 CDN 的规则源（任一可达即生效），并 `enable` 开机自启。WAN 未就绪时包自带的 procd 触发器会在联网后重试下载。之后在 **服务 → AdBlock-Fast** 可换源、加白名单、暂停过滤。

> 另两个官方选项未采用：经典 `adblock` 依赖 `firewall4` + `coreutils` + `gawk` 更重且配置项繁杂；`simple-adblock` 没有 LuCI 面板。想换回 AdGuardHome 只需在 `config.seed` 里把两行改回去，同时删掉 `98-adblock-fast`。

## 网络加速（ACC）：出厂已带

x86/64 上没有硬件 NAT / HNAT / SFE / NSS（这些是 MTK、高通芯片专用），能用的加速是内核自带的三项：

| 能力 | 编入方式 | 出厂默认 | 在哪调 |
|---|---|---|---|
| BBR 拥塞控制 | `kmod-tcp-bbr` | **开**（`tcpcca=bbr`） | 网络 → 防火墙 → 区域设置 → TCP CCA |
| FullCone NAT（全锥） | `kmod-nft-fullcone` | **开**（`fullcone=1`） | 同上 → Enable FullCone NAT |
| 软件 Flow Offload | `kmod-nft-offload` | 关 | 同上 → Flow offloading type |

`coolsnowwolf/lede` 的 `firewall4` 本身就把这三个 kmod 列为 `DEPENDS`，`coolsnowwolf/luci@openwrt-25.12` 的防火墙页面也已内置全部开关，所以**不需要 `luci-app-turboacc`**：它不在任何官方 feed 里，且是为旧版 iptables / `xt_FLOWOFFLOAD` 写的，与 nftables / firewall4 的键名和内核模块都不兼容。旧版 Turbo ACC 面板里的 DNS 缓存一项，本固件由 dnsmasq + MosDNS 承担；`luci-app-xlnetacc` 是迅雷快鸟宽带提速（要账号 + 运营商支持），与此无关。

默认参数由 `files/etc/uci-defaults/99-acc-tuning`（首次开机执行一次后自动删除）与 `files/etc/sysctl.d/90-acc-tuning.conf` 写入；工作流把它们复制进 `lede/files/`，OpenWrt 会原样叠加到 rootfs，刷完即生效，不用装包。

> **软件 Flow Offload 为什么默认关**：本固件带 SSR-Plus 透明代理与 SQM。flowtable 会让已建立连接绕过 nftables 规则，典型症状是「测速正常但代理分流失效」。不用代理 / SQM 时，在防火墙页面把 *Flow offloading type* 选成 `Software flow offloading` 即可，不用重编译、不用装包。硬件卸载在 x86 上永远保持关闭。

## 编译前请留意

- **首次编译会比较久**：33 个 LuCI 应用/主题里有十几个 Go 程序，全量编译可能 4–6 小时。`timeout-minutes` 已设 345（GitHub 单 job 硬上限 360），`dl/` 与 `.ccache` 已缓存，第二次起会快很多。
- **Docker 是唯一明显拉长编译的项**。若首次构建因它超时或失败，把 `config.seed` 里 `luci-app-dockerman` / `dockerd` / `docker` 三行注释掉再跑，其余不受影响。
- 刷机后**不需要再串联两个 DNS 进程**：adblock-fast 不占 53，MosDNS 保持原样即可。若发现广告没被拦，先到「服务 → AdBlock-Fast」看规则源是否下载成功，必要时换成国内可访问的列表。
- 软件 Flow Offload 默认关闭（与透明代理、SQM 互斥），开启方式见上节；x86 没有硬件 NAT，硬件卸载请勿开启。
- `config.seed` 里 `INCLUDE_*` 是 Kconfig `choice`，工作流 Step Summary 的「显式包未生效数」偶尔会列出它们，属正常，不阻断编译；`xray-core`、`mihomo` 已单独写入 seed 兜底。

## 关键改动（工作流）

- 独立 `config.seed`，一次写入 + 一次 `make defconfig`；逐项校验被 defconfig 静默丢弃的符号；显式声明 x86/64 target。
- 启用 `CONFIG_CCACHE=y`，用 `actions/cache` 缓存 `lede/dl` 与 `lede/.ccache`，缓存键按周轮换。定时由每日改每周。
- 单次 `make -j2`（标准 runner 2 核 / 7GB，`-j4` 易 OOM），删掉 `-j4 → -j2 → -j1` 盲重试；`timeout-minutes: 345`；`concurrency` 防重复并发。
- 不再吞错：去掉 `continue-on-error` 与满屏 `|| true`；三个产物上传全部 `if-no-files-found: error`；没有固件直接判失败；失败时上传 `build.log`。
- `free-disk-space` 用新仓库名 `lumbroso-commons/...` 与新参数 `preinstalled-runtimes`；**保留 swap**（`swap-storage: false`），避免内存压力下被静默杀掉；`upload-artifact` v7。
- 新增「注入出厂默认值（ACC + 去广告）」步骤：把 `files/` 下两个 uci-defaults 脚本与 sysctl.d 配置复制进 `lede/files/`，校验可执行位与关键行，随 rootfs 一起编进固件。

## 产物

3 个 artifact，保留 7 天：

- `OpenWrt_firmware_x86_64`：固件（`.img.gz` / `.img` / `.vmdk` 等）
- `OpenWrt_package_x86_64`：编译出的 ipk（保留原始目录结构）
- `OpenWrt_buildinfo_x86_64`：`.buildinfo` / `.manifest` / 校验和 / `profiles.json`

Lean 主线用 **opkg / ipk**，`openwrt-25.12` 只是 LuCI feed 分支名，别下 apk 格式的第三方包。

## 加插件

在 `config.seed` 里加一行 `CONFIG_PACKAGE_<包名>=y`，包名必须是官方 feed 里真实存在的目录名。改完手动触发一次，看 Step Summary 的「显式包未生效数」是否为 0（选项类符号的 warning 除外）。

`coolsnowwolf/luci` 的 `openwrt-25.12` 分支有 180 个应用，passwall2 / nikki / openclash / homeproxy / netdata / vnstat2 / msd_lite / udpxy 等都在，需要时开启即可，但注意不要再引入第二个透明代理面板。

## 本地重建 / 补丁

改动已推送到仓库 `main` 分支。若要在另一个 clone 里重建同一套文件：

```bash
git clone https://github.com/ngbaozi/openwrtBuild.git
bash apply.sh openwrtBuild       # 复制全部文件、删旧 bulid.yml、commit、push
```

`openwrtBuild.patch` 是**相对最初仓库（只有拼错的 `bulid.yml`）的完整补丁**，供从零套用；对已经改造过的仓库请改用 `apply.sh`：

```bash
cd openwrtBuild
git apply openwrtBuild.patch
git add -A && git commit -m "收敛到官方 feed，一次编入 + ACC + adblock-fast" && git push origin HEAD
```
