# 小米 R1350（小米路由器 4 Pro）OpenWrt + OpenClash 云编译方案

> 本仓库已配置好 GitHub Actions，自动编译 OpenWrt 24.10 **精简固件**，并把 **OpenClash 单独编译成 ipk** 一起发布（刷机后再安装）。
> 刷机请仔细阅读 **[刷机教程-R1350.md](./刷机教程-R1350.md)**（非常详细，照着做即可）。

> **为什么分开？** R1350 是 16MB flash，firmware 分区仅 14MB。OpenClash 的 `ruby/ruby-yaml` 依赖链约 10MB，编进固件会让镜像达 17.8MB、超出分区而无法刷入。拆成 ipk 后固件仅约 7-8MB，能 100% 正常刷入，OpenClash 按需安装。

## 0. 先确认设备

- 型号：**小米路由器 4 Pro**，型号 **R1350 / YA105**
- 主控：Qualcomm **QCA9563**（2.4G），5G 为 **QCA9888**（PCIe，`168c:0056`）
- 16 MB SPI NOR Flash / 128 MB RAM，AR8327 交换机（1 WAN + 3 LAN）
- 官方 2020-06 上市的 AC3000 路由器

> 注意：R1350 ≠「小米路由器 4」≠「小米路由器 Pro」。刷机前务必确认机身标签/系统内型号匹配，否则会变砖。

固件基于第三方移植仓库 `nyarime/openwrt-r1350`（OpenWrt 24.10），已在真机验证：2.4G/5G、网口、LED、复位键均可用；OpenClash 来自 `vernesong/OpenClash`。

## 1. 为什么用云编译

OpenWrt 必须在 Linux 下编译。GitHub Actions 免费提供 Ubuntu 云主机（每次 6 小时、每月 2000 分钟），足够编译这个 16MB 的小目标，且不占用本地资源。

## 2. 触发编译

本仓库 `.github/workflows/build-r1350.yml` 已配置完成，工作流名为 `Build OpenWrt for Xiaomi R1350 (Mi Router 4 Pro) + OpenClash IPK`。

进仓库「Actions」标签 → 左侧选该工作流 → 右侧「Run workflow」→ 确认即可。首次 push 也会自动触发一次。编译约 20–60 分钟，点进某次运行可实时看日志。

工作流做的事：
1. 用固定 commit 克隆 OpenWrt 24.10（保证可复现）
2. 复制 R1350 完整 DTS + 应用 image/LED 补丁（绕过上游 0001 补丁的格式 bug）
3. 注册 OpenClash feed；固件里只加小依赖（`dnsmasq-full`、`kmod-nft-tproxy` 等）
4. OpenClash 及其 ruby 依赖用 `=m` 只编译成 ipk、**不进固件**
5. 编译，并**检查固件是否超过 14MB 分区**
6. 把固件 + OpenClash 全套 ipk + 一键安装脚本打包，上传 Artifacts 并发布 Release

## 3. 拿固件

两种下载方式：
1. **Artifacts**：运行结束后页面底部 `openwrt-r1350-firmware-plus-openclash-ipk`（保存 30 天）。
2. **Releases**：仓库右侧 Releases，含 `SHA256SUMS` 校验文件，可长期保存。

产物文件：
- `...-squashfs-sysupgrade.bin` —— 精简固件（约 7-8MB，可正常刷入）
- `...-initramfs-kernel.bin` —— RAM 引导验证用（不写 flash）
- `openclash-ipk-bundle.tar.gz` —— OpenClash 全套离线安装包（含 `luci-app-openclash`、ruby 等依赖 ipk + 一键 `install.sh`），刷机后按刷机教程第十节安装
- `SHA256SUMS` —— 校验和

> 首次从原厂固件刷入，推荐先用 `R1350-initramfs-kernel.bin`（来自 `https://dl.naixi.net/openwrt/r1350/` 或移植仓库）做 **RAM 引导验证**，确认无线/网口/LED 正常后再写固件。详见刷机教程路线 A。

## 4. 刷机关键警告（详见刷机教程）

- **先备份整片 flash**：`all.bin`、`art.bin`、`bootloader.bin`、`firmware.bin`。
- **TTL 串口用 3.3V**，切勿接 5V。
- **ART 分区（0x060000, 64KB）绝对不能覆盖**，否则无线校准数据丢失、Wi-Fi 无法使用。恢复原厂时也只能用自己设备的备份写 `firmware` 分区（偏移 0x200000，最大 14MB），**不要写别人的 ART/Bdata/Nvram**。

分区布局：Bootloader 0x000000/192KB · Nvram 0x030000/64KB · Bdata 0x040000/64KB · crash 0x050000/64KB · ART 0x060000/64KB（只读） · cfg_bak 0x070000/128KB · firmware 0x200000/14MB。

## 5. 空间说明（重要）

固件已是精简版（约 7-8MB），远低于 14MB 分区，工作流的大小守卫会打印实际余量。OpenClash 单独以 ipk 安装，装完仍会占用不少 flash，建议：

- clash/mihomo 内核与规则集放 `/tmp`（128MB RAM 够用）或 U 盘，减少 flash 占用；
- 若 flash 吃紧，可改 extroot（U 盘扩展 overlay）后再装 OpenClash；
- 装内核前用 `df -h /overlay` 确认至少 15MB 可用空间。

## 6. 常见问题

- **`python3-distutils` 安装失败**：工作流已固定 `ubuntu-22.04`，正常不会出现。
- **`make download` 报网络错**：工作流已做二次单线程重试；个别源偶发失败可重跑，已下载文件命中缓存。
- **编译出的固件超限**：精简方案下不应出现（约 7-8MB）；若真出现见第 5 节。
- **安装 OpenClash ipk 报依赖冲突**：固件已预装 `dnsmasq-full`，必要时用 `opkg install --force-depends ./*.ipk`；详见刷机教程第十节。
- **刷机/OpenClash 配置问题**：见 [刷机教程-R1350.md](./刷机教程-R1350.md)。
