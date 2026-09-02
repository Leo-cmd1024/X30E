# X30E ImmortalWrt 云编译仓库

用 GitHub Actions 编译锐捷 RG-X30E 的 ImmortalWrt 固件。
仓库里**只有编译配置**（几 KB），源码由 runner 在境外拉取，绕开本机到 GitHub 的网络问题。

## 关键设定

| 项 | 值 | 说明 |
|---|---|---|
| 源码 | `RuijieNetworksCommunity/immortalwrt-mt798x` 分支 `openwrt-21.02` | 已内置 X30E 支持 |
| 目标设备 | `ruijie_rg-x30e-firmware2` | 专为刷入 **firmware2 备份分区**设计，原厂固件留在 firmware 分区不动 |
| 基础配置 | `defconfig/mt7981-ax3000.config` | 含 mt_wifi 闭源无线驱动 + hnat 硬件 NAT，**硬件加速不丢** |
| Runner | `ubuntu-22.04`，4 核 / 16GB / 免费不限时 | 公共仓库规格 |

## 建仓库并推送

```sh
cd openwrt/ci-repo
git init
git add -A
git commit -m "init: X30E immortalwrt build"

# 在 GitHub 网页上新建一个 Public 空仓库，然后：
git remote add origin git@github.com:<你的用户名>/<仓库名>.git
git branch -M main
git push -u origin main
```

若 SSH 推送不通，改用 HTTPS：

```sh
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
```

## 触发构建

推送后自动触发。也可手动：仓库页面 → **Actions** → `Build ImmortalWrt for Ruijie RG-X30E` → **Run workflow**。

构建约 40–90 分钟。

## 取固件

Actions 运行完成后 → 页面底部 **Artifacts** → `x30e-immortalwrt` → 下载。

需要的两个文件（名字含 `firmware2`）：

- `...-ruijie_rg-x30e-firmware2-squashfs-sysupgrade.bin` — sysupgrade 用
- `...-ruijie_rg-x30e-firmware2-squashfs-factory.bin` — 若需从原厂界面刷入则用这个

## 刷机前必读

1. **先备份**：`openwrt/backup/` 里的 `mtd3-factory.bin`（无线校准数据）已存本地，请另存一份到别处。丢了无法重建，WiFi 信号会废。
2. **这是一台在用的主路由**（pppoe 已拨号、有客户端在线），刷机会断网，挑时间。
3. **救砖保障**：uboot 有双分区自动回退。新固件起不来会自动爬回原厂分区，不需要串口。
4. **闭源加速仍在**：mtwifi + hnat + warp 都在这条分支里，NAT 吞吐和 WiFi 性能接近原厂，不是主线 OpenWrt 那种"加速全丢"的情况。
5. **镜像体积**：必须塞进 34.5MB 的 firmware2 分区。基础配置已接近上限，加插件前先确认空间。

## 加插件

编辑 `.github/workflows/build-x30e.yml`，在 `Build config` 步骤的 `{ ... } >> .config` 块里追加，例如：

```
CONFIG_PACKAGE_iperf3=y
CONFIG_PACKAGE_tcpdump=y
CONFIG_PACKAGE_luci-app-ddns=y
```

改完 push 即重新构建。
