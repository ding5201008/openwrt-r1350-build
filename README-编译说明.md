# 小米 R1350（小米路由器 4 Pro）OpenWrt + OpenClash 云编译方案

> 本仓库已配置好 GitHub Actions，会自动编译带 **OpenClash** 的 OpenWrt 24.10 固件。
> 刷机请仔细阅读 **[刷机教程-R1350.md](./刷机教程-R1350.md)**（非常详细，照着做即可）。

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

本仓库 `.github/workflows/build-r1350.yml` 已配置完成，工作流名为 `Build OpenWrt for Xiaomi R1350 (Mi Router 4 Pro) + OpenClash`。

进仓库「Actions」标签 → 左侧选该工作流 → 右侧「Run workflow」→ 确认即可。首次 push 也会自动触发一次。编译约 20–60 分钟，点进某次运行可实时看日志。

工作流做的事：
1. 用固定 commit 克隆 OpenWrt 24.10（保证可复现）
2. 应用 R1350 的 DTS / image 补丁
3. 注册 OpenClash feed，把 `dnsmasq` 换成 `dnsmasq-full`，打开 `luci-app-openclash` 及 nftables 依赖
4. 编译，并**检查固件是否超过 14MB 分区**（超限会明确报错）
5. 上传固件到 Artifacts 并发布 Release

## 3. 拿固件

两种下载方式：
1. **Artifacts**：运行结束后页面底部 `openwrt-r1350-openclash-firmware`（保存 30 天）。
2. **Releases**：仓库右侧 Releases，含 `SHA256SUMS` 校验文件，可长期保存。

产物文件：
- `...-squashfs-sysupgrade.bin` —— 最终固件
- `luci-app-openclash_*.ipk` —— 单独编译的 OpenClash 安装包（备用，供 extroot / 精简固件按需安装）
- `SHA256SUMS` —— 校验和

> 首次从原厂固件刷入，推荐先用 `R1350-initramfs-kernel.bin`（来自 `https://dl.naixi.net/openwrt/r1350/` 或移植仓库）做 **RAM 引导验证**，确认无线/网口/LED 正常后再写固件。详见刷机教程路线 A。

## 4. 刷机关键警告（详见刷机教程）

- **先备份整片 flash**：`all.bin`、`art.bin`、`bootloader.bin`、`firmware.bin`。
- **TTL 串口用 3.3V**，切勿接 5V。
- **ART 分区（0x060000, 64KB）绝对不能覆盖**，否则无线校准数据丢失、Wi-Fi 无法使用。恢复原厂时也只能用自己设备的备份写 `firmware` 分区（偏移 0x200000，最大 14MB），**不要写别人的 ART/Bdata/Nvram**。

分区布局：Bootloader 0x000000/192KB · Nvram 0x030000/64KB · Bdata 0x040000/64KB · crash 0x050000/64KB · ART 0x060000/64KB（只读） · cfg_bak 0x070000/128KB · firmware 0x200000/14MB。

## 5. 空间说明（重要）

OpenClash 依赖 `ruby` / `ruby-yaml` 等，体积较大，而 R1350 的 `firmware` 分区只有 14MB。工作流已内置大小检查；若报 `Firmware exceeds 14 MiB`：

- 在 `Prepare .config with OpenClash` 步骤里去掉非必要软件包；
- 或改用 extroot（U 盘扩展 overlay）后单独 `opkg install` 附带的 OpenClash ipk；
- OpenClash 内核与规则集建议放 `/tmp` 或 U 盘，减少 flash 占用。

## 6. 常见问题

- **`python3-distutils` 安装失败**：工作流已固定 `ubuntu-22.04`，正常不会出现。
- **`make download` 报网络错**：工作流已做二次单线程重试；个别源偶发失败可重跑，已下载文件命中缓存。
- **编译出的固件超限**：见第 5 节空间说明。
- **刷机/OpenClash 配置问题**：见 [刷机教程-R1350.md](./刷机教程-R1350.md)。
