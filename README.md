# PonWrt 6.18 for 烽火 HG5585F (Airoha AN7581) — chkdsk228 定制版

简体中文 | [English](README_en.md)

[![GitHub Actions Build Status](https://img.shields.io/github/actions/workflow/status/chkdsk228/ponwrt-ZN504-HG5585F/build-zn504.yml?branch=master&style=flat-square&logo=github)](https://github.com/chkdsk228/ponwrt-ZN504-HG5585F/actions)
[![Latest Release](https://img.shields.io/github/v/release/chkdsk228/ponwrt-ZN504-HG5585F?style=flat-square&logo=github)](https://github.com/chkdsk228/ponwrt-ZN504-HG5585F/releases)
[![Target](https://img.shields.io/badge/SoC-Airoha%20AN7581-blue?style=flat-square)](https://github.com/chkdsk228/ponwrt-ZN504-HG5585F)
[![Device](https://img.shields.io/badge/Device-HG5585F%20CT%20%26%20CU-green?style=flat-square)](https://github.com/chkdsk228/ponwrt-ZN504-HG5585F)
[![Kernel](https://img.shields.io/badge/Kernel-Linux%206.18-informational?style=flat-square)](https://github.com/chkdsk228/ponwrt-ZN504-HG5585F)
[![License](https://img.shields.io/badge/License-GPL--2.0-orange?style=flat-square)](LICENSE)

本项目基于 **[wuziwei/ponwrt-ZN504-HG5585F](https://github.com/wuziwei/ponwrt-ZN504-HG5585F)**（其上游为 [PonWrt](https://github.com/pbs05/ponwrt)，ImmortalWrt / Linux `6.18`）的完整源码树。在**保留上游全部硬件级优化与固件配置（不做任何精简）**的前提下，追加 **chkdsk228 的四项个人定制**，并针对烽火 HG5585F 双变体做构建适配：

* **烽火 HG5585F 电信版 (`fiberhome_hg5585f-ct`)**：带 Wi-Fi 6 光猫（AN7581 + MT7916D DBDC AX3000 + 4 核中断亲和智能调度，lan1 为 GPY211C 2.5G 独立 PHY）；
* **烽火 HG5585F 联通版 (`fiberhome_hg5585f-cu`)**：同平台不同 DTS（lan1-4 全部走内部 PHY 千兆，LED 为绿色）。

每次发布均**同时自动化编译并提供两款变体的固件包**！

---

## 🎯 零、chkdsk228 定制项（本项目相对上游的唯一差异）

> **设计原则**：上游的一切（源码树、`configs/`、`files/`、`scripts/`）**原样保留，不做精简**。本项目只追加四项定制，其余与上游 `wuziwei/ponwrt-ZN504-HG5585F` **完全一致**。

| # | 定制项 | 上游默认值 | 本项目值 | 注入位置 |
| :--- | :--- | :--- | :--- | :--- |
| **1** | 设备主机名 | `ImmortalWrt`（系统默认） | **`OWRT`** | `files/etc/uci-defaults/98-chkdsk228-custom` |
| **2** | 后台管理地址 | `192.168.1.1`（上游未显式设置） | **`192.168.10.1`** | 同上 |
| **3** | 构建者标识 | 无 | **`build by chkdsk228`**（显示于 LuCI 概览页版本行） | 同上（`sed` 注入 `10_system.js`） |
| **4** | 固件命名与 Release tag | `ponwrt-airoha-an7581-<设备>-squashfs-sysupgrade.itb`<br>tag: `v2026.11.17-0850`（精确到分） | **`immortalwrt-<机型>-squashfs-enhanced-<YY.MM.DD>-<HH.MM.SS>.itb`**<br>tag: **`v<YY.MM.DD>-<HH.MM.SS>`**（精确到秒，避免同日覆盖） | `.github/workflows/build-zn504.yml` |

### 定制实现细节

**定制 1-3** —— `files/etc/uci-defaults/98-chkdsk228-custom`（权限 `0755`，`98-` 前缀保证在 `99-custom-settings` 之前执行）：

```sh
#!/bin/sh
# 1. 设备主机名
uci -q set system.@system[0].hostname='OWRT'
uci -q commit system

# 2. 后台地址
uci -q set network.lan.ipaddr='192.168.10.1'
uci -q set network.lan.netmask='255.255.255.0'
uci -q commit network

# 3. 构建者标识 build by chkdsk228
for JS in /www/luci-static/resources/view/status/10_system.js \
          /usr/share/luci/menu.d/10_system.js; do
    [ -f "$JS" ] && sed -i "s/(\(luciversion || ''\))/(\1) + (' \/ build by chkdsk228')/g" "$JS" 2>/dev/null || true
done

exit 0
```

**定制 4** —— `.github/workflows/build-zn504.yml` 的 `Organize output files` 步骤新增重命名循环：

```sh
STAMP="$(TZ=Asia/Shanghai date +'%y.%m.%d-%H.%M.%S')"
for f in *.itb; do
  [ -f "$f" ] || continue
  case "$f" in
    *-initramfs-recovery.itb)   KIND="initramfs";        DEV="${f%-initramfs-recovery.itb}" ;;
    *-squashfs-sysupgrade.itb)  KIND="squashfs-enhanced"; DEV="${f%-squashfs-sysupgrade.itb}" ;;
    *) continue ;;
  esac
  MODEL="${DEV##*-an7581-}"
  MODEL="${MODEL#fiberhome_}"; MODEL="${MODEL#nokia_}"
  MODEL="${MODEL#znxt_}";      MODEL="${MODEL#gemtek_}"; MODEL="${MODEL#unionman_}"
  mv -v "$f" "immortalwrt-${MODEL}-${KIND}-${STAMP}.itb"
done
```

`Generate release tag` 步骤同步改为 `echo "release_tag=v${FW_STAMP}"`。

---

## 💡 一、上游与公版的核心差异对比矩阵

> 以下内容为上游 `wuziwei/ponwrt-ZN504-HG5585F` 的成果，本项目**完整继承**，未作任何删减。

| 核心功能 / 优化维度 | 上游公版 & 社区现状 (痛点) | 本源码树聚合增强 (独家根治与实测收益) |
| :--- | :--- | :--- |
| **IPTV 1302 鉴权根治** | ❌ 组播口与业务口共用网桥，动态 MAC 学习引发 FDB 翻转单播黑洞；社区多仅重启或转单播 | ✅ **独家补丁 `03-pon-userspace-iptv`**：组播口 `learning=0` + 动态探测业务口 BNG MAC 并静态固化，**彻底根除 1302 超时与黑洞，切台秒开** |
| **HG5585F 满血 Wi-Fi 6** | ❌ 缺 EEPROM 提取与射频校准；软中断挤死 Core 0（跑满即 100% 单核假死），测速仅 500~670M | ✅ **MT7916D 满血驱动 + 独家 4 核中断亲和调度**：`pin-wifi-cpus.sh` 将收发与 NAPI 线程精准均衡至 4 核，突破单核瓶颈，**实测 5G 突破 770~1100+ Mbps**，软中断零积压 |
| **PON 参数跨固件防丢** | ❌ 升级固件默认冲刷覆盖 `/etc/config/pon`，未手动保存直接失联变砖；社区需登录 SSH 手动备份 | ✅ **内置一键备份/还原脚本**，开机自动生成并注入系统升级白名单，**无感保留 LOID、SN、MAC 与宽带账号**，升级零丢配置 |
| **NNdroid OLT 局端防掉线** | ❌ 缺激光冷机防抖，偶发误报 `SYS_TX_READY` 被局端秒踢；华为等 OLT 报 `reason=0x01` 难以定位 | ✅ **集成 EN7572 激光冷启动 8 次采样闭锁保护**；支持 Web 自由切换 LOID 补零；状态页直显 16 进制报文，**冷启动 100% 稳出光** |
| **QWRT 16MB 网络缓冲** | ⚠️ 内核默认 2MB 缓冲，队列深度低，高并发 BT/多任务时交换芯片 CDM 队列易溢出丢包 | ✅ **对齐 QWRT 16MB (`16777216`) TCP/UDP 核心缓冲**，队列深度扩至 16384，内存防碎片，**彻底消除 CDM 突发溢出丢包** |
| **广播免跟踪 (NOTRACK)** | ❌ UPnP SSDP 与 Windows WSD 广播泛滥进入 conntrack 状态表，导致查表开销剧增与队列膨胀 | ✅ **预置 SSDP (`239.255.255.250:1900`) 与 WSD (`3702`) 全链路 `NOTRACK` 规则**，阻断局域网广播风暴污染连接跟踪表 |
| **开箱即用 br-iptv 代理** | ❌ 无预置网桥与代理，需自行抓包建桥；旧版单栈 udpxy 易引发二层组播环路反射与端口占用 | ✅ **预置隔离网桥 `br-iptv`（`isolate=1, learning=0`）**，集成 `rtp2httpd` 端口 4022 多核并发零拷贝，附带 Web 播放器，**全屋网页秒开** |
| **双栈硬件流卸载 (PPE)** | ⚠️ 常见固件缺少网桥卸载补丁导致下行卸载率仅 17%；IPv6 PPPoE 严重丢包卡死，迫使用户关闭加速 | ✅ **独家补丁 931/932 修复 AN7581 PPE 缺陷与网桥从端口卸载**：实现 IPv4+IPv6 双栈 100% 直转，**千兆满速 CPU 仅 ~4%，下行硬件卸载率飙升至 90%+** |
| **纯净硬件加解密优化** | ⚠️ 保留 SafeXcel / EIP-93 协处理器驱动，诱发 Crypto API 优先级霸凌、DMA 性能倒挂与锁争用 | ✅ **彻底剔除 `kmod-crypto-hw-eip93` 负优化**，释放 4 核 1.2GHz ARMv8 NEON 向量直算，**小包 DMA 刷脏归零，WireGuard/Sing-Box 吞吐更稳** |
| **Sing-Box 原生 NFQUEUE & Honk eBPF** | ❌ 缺失 `kmod-nft-queue` 与 `kmod-veth` 驱动，开透明代理必崩溃；依赖脆弱易冲突的自定义脚本 | ✅ **内核原生集成 `kmod-veth`、NFQUEUE 与最新 `sing-box 1.15.0-alpha.13` + `honk 0.1.0-beta.6`**，**原生 TUN 模式与 eBPF 零拷贝双核透明网关** |
| **现代化 CI/CD 流水线** | ⚠️ 基础单任务无编译器指纹匹配，依赖易超时，编译动辄 2~4 小时且网络波动易断流 | ✅ **Ubuntu 24.04 + Node 24 + CCACHE 编译器指纹匹配**，5GB 增量编译缓存与多镜像测速回退，**构建大幅缩短至 20~35 分钟，成功率 99%+** |

---

## 🚀 二、上游独家新功能与深度优化拆解

### 1. 🌟 电信 IPTV 1302 鉴权超时与网桥 MAC 表翻转黑洞

* **二层组播口 `learning=0` 静态化**：`br-iptv` 网桥成员 `pon0.50`（组播 VLAN）关闭 MAC 学习，杜绝上游 FDB 表项被组播流量翻转覆盖导致的单播黑洞。
* **业务口 BNG MAC 动态探测与静态固化**：`/etc/hotplug.d/iface/95-iptv-m3u-ip` 在 IPTV 接口上线时探测 BNG 侧 MAC 并写入静态 FDB，避免 1302 鉴权超时。
* **`isolate=1` 端口隔离**：`pon0.45`（IGMP 上行）与 `pon0.50`（组播下行）双向隔离，防止二层组播环路反射。

### 2. ⚡ 烽火 HG5585F Wi-Fi 6 满血驱动与独家 4 核中断亲和调度

* **MT7916D 满血驱动与 NPU 辅助支持**：完整预置 `en7581_MT7916_npu_rv32.bin` 与 `en7581_MT7916_npu_data.bin` 等 NPU 辅助固件。针对当前开源主线 AN7581 与 mt76 尚未打通纯硬件 WED 直转的现状，独创采用四核智能中断亲和架构，彻底破解开源公版"单核软中断被打满、测速被卡在 500~600M"的瓶颈。
* **工厂 EEPROM 校准自动提取**：内置 `/etc/hotplug.d/firmware/10-mt7916-an-eeprom` 与 `/etc/init.d/mt7916-an-eeprom`（在 `kmodloader` 加载驱动前于 `START=08` 执行），开机自动从 UBI `factory` 分区的 `0x1000` 偏移处提取 4KB 射频校准数据，校验 `0x7916` 硬件魔数并无缝提供给驱动。
* **独立 MAC 地址算号机制**：内置 `/lib/preinit/07_an7581_bsp_wifi_mac`，开机预初始化阶段自动从 `factory` 分区读取基准 MAC 并计算 2.4G (`base + 6`) 与 5G (`base + 7`) 独立合法硬件地址。
* **射频功率校准表**：预置 `SingleSKU` 与 `mt7916-sku.dat` / `mt7916-sku-bf.dat` 波束成形数据。
* **Wi-Fi 四核收发线程精准绑核**（`/usr/sbin/pin-wifi-cpus.sh` + `/etc/hotplug.d/net/99-wifi-affinity`）：
  - 发送线程 `mt76-tx*` 独占绑定至 `CPU3`；
  - 6 个 `napi/phy0-0` 收包线程按 `0,0,0,1,2,2` 均衡分布至 `CPU0 / CPU1 / CPU2`；
  - 有线以太网硬中断精准绑定：`IRQ 17 -> CPU1`、`IRQ 21 -> CPU2`；
  - 实测 HG5585F 5G Wi-Fi iPerf3 5 线程稳定 **1144 Mbps 下行 / 1200 Mbps 上行**（峰值 **1314+ Mbps**）。
* **异步无线扫描防卡死机制**：集成 `rpcd` 异步信道扫描补丁（`133-rpcd-async-scan.patch`），将长耗时的 Wi-Fi 信道扫描任务从主事件循环解耦，杜绝在 LuCI 点击扫描时造成的 Web 控制台无响应与 504 Gateway Timeout。

### 3. 🛡️ PON 认证参数与透明代理订阅跨固件升级自动备份还原

* **一键提取与恢复脚本**：内置 `/usr/sbin/backup-fiberhome-pon.sh` 与 `/usr/sbin/restore-fiberhome-pon.sh`，一键提取并校验 `FHTT`、`HG5585FV1.0`、LOID、PON SN 及设备标识。
* **开机自动守护**：首次启动或配置变更时，系统自动维护 `/etc/sysupgrade.conf` 白名单，将 `/etc/config/pon`、`/etc/fiberhome_pon_backup.uci`、Sing-Box/Honk 配置文件与管理脚本永久纳入保护清单。
* **升级零丢失**：无论通过 Web 界面还是 SSH 命令行执行 `sysupgrade`，原有的运营商认证信息与透明代理规则均完整保留。

### 4. 🔍 特殊局端 OLT（华为/中兴）认证兼容性增强

* **达发 EN7572 激光冷启动防抖与闭锁保护**：集成 8 次连续硬件采样确认（140ms 稳定期），彻底修复光猫冷启动上电瞬间激光器未完全出光而偶发误报 `SYS_TX_READY` 导致被局端 OLT 秒踢下线的顽疾。
* **LOID 补零方向与原始报文回显**：Web 界面可自由切换 LOID 填充方向（左补零/右补零），状态页直显 16 进制原始报文，便于与局端 OLT 侧日志比对定位。

### 5. 🚀 QWRT 16MB 网络核心缓冲区、16384 netdev 队列与页面防碎片分配

* **16MB 大缓冲**：`/etc/sysctl.d/30-network-buffers.conf` 设置 `net.core.rmem_max/wmem_max=16777216`，`tcp_rmem/tcp_wmem` 第三列对齐 16MB，适应万兆长肥管道（BDP）。
* **队列深度扩容**：`netdev_max_backlog=16384`、`netdev_budget=600`、`dev_weight=128`，消除交换芯片 CDM 队列突发溢出。
* **页面防碎片**：`vm.percpu_pagelist_high_fraction=128`，约束多核本地空闲页缓存上限，防止 512MB 小内存下的内存碎片与饥饿。

### 6. 📺 开箱即用 br-iptv 异 VLAN 端口隔离桥接与 rtp2httpd (4022) 双栈组播转单播

* **预置隔离网桥**：`br-iptv` 绑定 `pon0.45` / `pon0.50`，`igmp_snooping=0`，成员 `isolate=1`。
* **rtp2httpd 预调优**：监听 `4022`，`workers=4`，`maxclients=20`，`buffer_pool_max_size=16384`，`udp_rcvbuf_size=2097152`，绑定 `br-iptv` 上行。
* **老组件自动禁用**：检测到 `msd_lite` / `udpxy` 时自动 `enabled=0`，避免端口冲突。
* **播放列表自适配**：首次启动自动将 `/www/iptv.m3u` 中的地址同步为当前 LAN IP。

### 7. ⚙️ 现代化 CI/CD 构建流水线

* **Ubuntu 24.04 + CCACHE 编译器指纹匹配**：`CCACHE_COMPILERCHECK='%compiler% -dumpmachine; %compiler% -dumpversion'`，`CCACHE_MAXSIZE=5G`。
* **三档缓存模式**：手动触发时可选择「使用缓存 / 重新编译 / 不使用缓存」。
* **下载缓存**：`dl` 目录按 `feeds.conf.default` + `configs/*.config` 哈希缓存。
* **编译重试链**：`make -j` 失败自动降级 `make -j1 V=s`，输出完整错误日志。
* **产物完整性**：自动生成 `.sha256` 校验文件。

---

## 📦 固件下载 (Releases)

请前往 [GitHub Releases](https://github.com/chkdsk228/ponwrt-ZN504-HG5585F/releases) 获取固件包。

**命名规则**：
```
immortalwrt-<机型>-squashfs-enhanced-<YY.MM.DD>-<HH.MM.SS>.itb   # 系统升级
immortalwrt-<机型>-initramfs-<YY.MM.DD>-<HH.MM.SS>.itb           # 恢复/救砖
```

**示例**：`immortalwrt-hg5585f-ct-squashfs-enhanced-26.11.17-22.09.39.itb`

| 适用设备 | 固件类型 | 适用场景 |
| :--- | :--- | :--- |
| **烽火 HG5585F (CT 电信版)** | `squashfs-enhanced` | **系统平滑升级（推荐）**，保留全部配置 |
| **烽火 HG5585F (CT 电信版)** | `initramfs` | **U-Boot Web 恢复/救砖** |
| **烽火 HG5585F (CU 联通版)** | `squashfs-enhanced` | **系统平滑升级（推荐）**，保留全部配置 |
| **烽火 HG5585F (CU 联通版)** | `initramfs` | **U-Boot Web 恢复/救砖** |

> **CT / CU 区分**：外壳标识为电信版选 CT，联通版选 CU。两者硬件同平台但网口拓扑与 LED 颜色不同（CT 蓝灯、CU 绿灯），**不可互刷**。

**固件体积**：约 65 MB（含上游全部功能：IPTV、Sing-Box、Honk、Footstrap 主题、Diskman、HQOS 等）。

---

## 🛠️ 刷机与升级指南

> [!IMPORTANT]
> **关于光猫认证参数保护（特别提醒）**：
> 接入运营商网络通常使用的是**克隆的原电信烽火（FiberHome / FHTT）光猫认证参数**（包含 `vendor_id='FHTT'`、`hardware_version='HG5585FV1.0'`、LOID 与 SN）。
> 升级前**切勿清空配置恢复出厂**，应直接使用保留配置升级，或在终端执行 `/usr/sbin/backup-fiberhome-pon.sh` 生成认证备份！

### 方式一：SSH 命令行保留配置安全升级（强烈推荐）

```bash
# 1. 将 sysupgrade 固件上传至光猫的 /tmp 目录
scp immortalwrt-hg5585f-ct-squashfs-enhanced-*.itb root@192.168.10.1:/tmp/

# 2. SSH 登录并校验固件完整性
ssh root@192.168.10.1
cd /tmp && sha256sum -c immortalwrt-hg5585f-ct-squashfs-enhanced-*.itb.sha256

# 3. 执行保留配置刷入（切勿使用 -n 参数，否则会清除所有参数）
sysupgrade -v /tmp/immortalwrt-hg5585f-ct-squashfs-enhanced-*.itb
```

### 方式二：LuCI Web 升级

登录 `http://192.168.10.1` → 系统 → 备份/升级 → 刷写固件 → 选择 `.itb` 文件 → 刷写。

### 方式三：U-Boot Web 恢复刷机

断电 → 按住 reset 键上电 → 保持 5 秒 → 浏览器访问 `http://192.168.1.1` → 上传 `initramfs` 固件 → 刷写。

---

## 📖 烽火 HG5585F PON 上线与 OLT 注册通关指南

### 1. 换猫前必须提取的"原光猫 4 要素"

| 要素 | 获取方式 | 用途 |
| :--- | :--- | :--- |
| **LOID** | 原光猫 Web 界面 / 运营商 APP | 局端 OLT 认证凭据 |
| **LOID 密码** | 同上 | 部分局端要求 |
| **PON SN** | 原光猫标签 / Web 界面 | 光路标识 |
| **Vendor ID / 硬件版本** | 原固件 `/etc/config/pon` | 局端设备类型匹配 |

### 2. 核心避坑实录

| 高频翻车点 | 原因 | 解决 |
| :--- | :--- | :--- |
| **LOID 认证失败** | 补零方向不匹配 | Web 界面切换 LOID 填充方向（左补零/右补零） |
| **注册成功但无法拨号** | VLAN 不匹配 | 从原光猫抓取 VLAN ID 并配置到 WAN |
| **频繁掉线** | 激光冷启动未稳 | 已内置 8 次采样防抖（见 §二.4） |
| **升级后认证丢失** | `sysupgrade` 冲刷配置 | 已内置自动备份（见 §二.3），或手动执行 `backup-fiberhome-pon.sh` |

### 3. 链路诊断与成功标志

```bash
# 查看 PON 守护进程状态
/etc/init.d/airoha-pon status

# 查看光模块收发光功率
ponctl -s

# 确认已进入 O5 状态（正常注册）
dmesg | grep -i "pon.*O5"
```

---

## 📱 三、客户端入口与默认凭据

| 项目 | 地址 / 值 |
| :--- | :--- |
| **LuCI 管理后台** | `http://192.168.10.1` |
| **设备主机名** | `OWRT` |
| **默认账号 / 密码** | `root` / `password`（首次登录请立即修改） |
| **IPTV 播放列表** | `http://192.168.10.1/iptv.m3u` |
| **rtp2httpd 状态页** | `http://192.168.10.1:4022/status` |

---

## 🏗️ 四、源码编译指南

### 1. 准备环境

推荐 Ubuntu 22.04 / 24.04 LTS：
```bash
sudo bash -c 'bash <(curl -s https://build-scripts.immortalwrt.org/init_build_environment.sh)'
```

### 2. 克隆代码与更新 Feeds

```bash
git clone https://github.com/chkdsk228/ponwrt-ZN504-HG5585F.git
cd ponwrt-ZN504-HG5585F

./scripts/feeds update -a
./scripts/feeds install -a
bash ./scripts/customize.sh
```

### 3. 应用 AN7581 增强配置

```bash
./scripts/kconfig.pl + configs/an7581.config configs/release.config > .config
make defconfig
```

### 4. 开始多线程编译

```bash
make download -j$(nproc)
make -j$(nproc) || make -j1 || make -j1 V=s
```

固件输出目录：`bin/targets/airoha/an7581/`

### 5. GitHub Actions 云编译（推荐）

Fork 本仓库 → Actions → `Build PonWrt (ZN504XG-D & HG5585F)` → Run workflow。

| 输入项 | 说明 |
| :--- | :--- |
| `upload_release` | 是否上传到 Release（默认 `true`） |
| `cache_mode` | `使用缓存` / `重新编译` / `不使用缓存` |

**触发条件**：手动触发，或 `configs/` `files/` `scripts/` `target/` `package/` 任一目录有变更时自动触发。

---

## 📁 五、仓库结构

```
ponwrt-ZN504-HG5585F/
├── .github/workflows/
│   ├── build-zn504.yml          # 主构建流程（本项目已加入时间戳命名定制）
│   └── release.yml              # Release 辅助流程
├── configs/
│   ├── an7581.config            # AN7581 完整目标配置（339 KB）
│   ├── an7583.config            # AN7583 完整目标配置（335 KB）
│   ├── release.config           # 发布包配置（84 项追加）
│   └── temperature-rpc.uc       # 温度 RPC 注入（不依赖 autocore ACL）
├── files/                       # rootfs overlay
│   ├── etc/
│   │   ├── uci-defaults/
│   │   │   ├── 98-chkdsk228-custom   # ★ 本项目定制（hostname/IP/build mark）
│   │   │   └── 99-custom-settings    # 上游网络/防火墙/IPTV/代理调优
│   │   ├── hotplug.d/           # WiFi 亲和性、PPPoE RPS、组播、IPv6 修复
│   │   ├── sysctl.d/            # 网络缓冲、BT conntrack 调优
│   │   ├── Wireless/mediatek/   # MT7916 SKU 射频校准表
│   │   └── config/              # Vantage 主题默认配置
│   ├── lib/firmware/airoha/     # MT7916 NPU 辅助固件
│   ├── usr/sbin/                # PON 备份/还原、WiFi 绑核脚本
│   └── www/                     # Vantage 主题资源、IPTV 播放列表
├── patches/                     # PON 驱动补丁（激光防抖、LOID、IPTV MAC）
├── scripts/
│   ├── customize.sh             # 自定义包集成（NPU/温度/主题/补丁）
│   ├── integrate-temperature.py # 温度 RPC 集成
│   └── kconfig.pl               # 配置合并工具
├── target/                      # 内核源码与 DTS
├── package/                     # 软件包
└── README.md
```

---

## ⚠️ 六、免责声明

1. 本项目为开源固件项目，旨在用于技术交流与网络设备功能研究。
2. 刷写第三方固件具有一定风险，操作前请务必使用 `backup-fiberhome-pon.sh` 备份好克隆的烽火 PON 认证数据（LOID、SN、Vendor ID）与宽带认证信息。
3. 请严格遵守当地电信运营商的相关管理规定，严禁用于任何非法网络活动。
4. 本项目基于 `wuziwei/ponwrt-ZN504-HG5585F` 二次定制，**上游功能的可用性与稳定性由上游保证**；本项目仅对四项定制负责。

---

## 🙏 七、引用致谢 (Acknowledgements & Credits)

- **[wuziwei/ponwrt-ZN504-HG5585F](https://github.com/wuziwei/ponwrt-ZN504-HG5585F)**：**本项目直接上游**，提供完整源码树、硬件级优化、构建流水线与固件配置。本项目仅在其基础上追加四项个人定制。
- **[pbs05/ponwrt](https://github.com/pbs05/ponwrt)**：PonWrt 创始人，奠定达发 Airoha AN7581 / AN7583 PON 光猫开源架构与驱动的坚实底座，本项目核心源码基石。
- **[1018ji/build-ponwrt-daily](https://github.com/1018ji/build-ponwrt-daily)**：现代化 CI 构建流水线开创者，提供基于编译器版本指纹的 Ccache 高速缓存持久化（`CCACHE_COMPILERCHECK`）、5GB 增量缓存复用与镜像源动态测速回退的架构灵感。
- **[LeoooChen/PonWRT-TYFZA1](https://github.com/LeoooChen/PonWRT-TYFZA1)**：硬件流量转发与 16MB 环形缓冲区调优先驱，贡献 XG-PON 线路与 PPPoE 线速流量硬件卸载实践及 `rpcd` 异步无线扫描防卡死机制。
- **[Kahen/PonWrt-CI-UBI](https://github.com/Kahen/PonWrt-CI-UBI)**：提供 AN7581 SoC 精确温度遥测方案 (`temperature-rpc.uc` 与 LuCI RPC 联动) 及 MTD 读写保护 (`kmod-mtd-rw`)。
- **[ImmortalWrt](https://github.com/immortalwrt/immortalwrt) & QWRT 项目团队**：为 Linux 网络内核调优、EIP-93 硬件协处理器剔除性能倒挂实践、PPE 硬件流表加速与网络调优提供深度参考与优秀范例。
- **[NNdroid](https://github.com/NNdroid)**：提供 EN7572 激光冷启动防抖确认补丁、LOID 填充方向调节及 CTC 原始报文分析回显工具链。
- **[OpenWrt Project](https://github.com/openwrt/openwrt)**：为全球网络设备提供坚实、可扩展的嵌入式 Linux 操作系统底座。
- **[stackia/rtp2httpd](https://github.com/stackia/rtp2httpd)**：高性能零拷贝 4K/8K IPTV 组播转单播、FCC 快速切台与 RTSP 代理引擎。
- **[SagerNet/sing-box](https://github.com/SagerNet/sing-box)**：新一代高性能网络代理核心。
- **[daeuniverse/honk](https://github.com/daeuniverse/honk)** 与 **[daeuniverse/openwrt-packages](https://github.com/daeuniverse/openwrt-packages)**：高性能 Linux eBPF 零拷贝透明代理引擎与 LuCI Web 控制面板。
- **[rchen14b/luci-app-airoha-npu](https://github.com/rchen14b/luci-app-airoha-npu)**：达发 AN7581 独立 8 核 RISC-V NPU / PPE 运行状态与硬件流表监控看板。
- **[VizzleTF/luci-theme-footstrap](https://github.com/VizzleTF/luci-theme-footstrap)**：专为 PonWrt 适配的清爽现代响应式主题。
- **[anothaDev/luci-theme-vantage](https://github.com/anothaDev/luci-theme-vantage)**：Vantage 主题本体与实况面板。
- **[oenhu](https://github.com/oenhu)**：贡献 PR #116 优化，避免 DHCP 网络冗余生成 `wan6` 接口引起的路由竞争。

---

## 🔗 相关链接

- [本项目 Releases](https://github.com/chkdsk228/ponwrt-ZN504-HG5585F/releases)
- [直接上游：wuziwei/ponwrt-ZN504-HG5585F](https://github.com/wuziwei/ponwrt-ZN504-HG5585F)
- [源码基石：pbs05/ponwrt](https://github.com/pbs05/ponwrt)
- [ImmortalWrt 官方文档](https://immortalwrt.org/)
