# BE72 Pro 云编译（GitHub Actions）

给 **锐捷 RG-BE72 Pro**（MediaTek MT7988DV / Filogic 860，3×A73 @1.8G / 1GB DDR4 / 256MB SPI-NAND / WiFi7 / 9 网口 / USB3.0）做两个云编译产物：

1. **第三方 U-Boot（FIP）** —— 解锁设备的关键，得先刷它
2. **ImmortalWrt 固件包（sysupgrade）** —— 真正释放性能的那一层

原厂 ReyeeOS 把设备锁死了：overlay 只剩 58MB + 私有 opkg 源，第三方插件一个都装不上。刷完 ImmortalWrt 后 overlay 到 232.5MB，playbook 里 90% 的玩法才能落地。

---

## 快速开始

### 第 1 步：编译 U-Boot（FIP）

```
Actions → "Build BE72 Pro U-Boot (FIP)" → Run workflow
```

参数：
| 参数 | 说明 |
|---|---|
| `multi_layout` = `2` | 两个变体都编（推荐，几分钟就完） |

也可以直接 push 触发 —— 改动 `.github/workflows/build-be72pro-uboot.yml` 或 `be72pro/uboot/**` 会自动跑。

产出（在 Release 和 Artifacts 里）：

| 文件 | 大小 | 配套固件 | 分区样式 |
|---|---|---|---|
| `mt7988_ruijie_be72_pro-fip-fixed-parts.bin` | ~1MB | `ruijie_rg-be72-pro`（**非 mod**） | ubi 保持 112.5M，最保守 |
| `mt7988_ruijie_be72_pro-fip-fixed-parts-multi-layout.bin` | ~1MB | `ruijie_rg-be72-pro-mod` | 可选 `ubi-232.5m`，闪存利用最大化 |

另外会顺带产出 `*-bl2.bin`，**别刷它**（见下面「为什么只刷 FIP」）。

### 第 2 步：编译固件

```
Actions → "Build BE72 Pro Firmware (ImmortalWrt)" → Run workflow
```

参数：
| 参数 | 说明 |
|---|---|
| `device` | `ruijie_rg-be72-pro-mod`（配 multi-layout FIP，overlay 232.5MB）/ `ruijie_rg-be72-pro`（配 fixed-parts FIP）/ `ruijie_rg-be72-pro-dsa-mod` |
| `with_diy` | `true` = 套用 diy.sh 定制（argon 主题、lucky、openlist2、nikki 代理、ttyd 免密、默认 IP 改 192.168.2.1） |

⚠️ **一次 2–6 小时**，别设 push 自动触发。建议先在 `be72pro/firmware/diy.sh` 里按需增删插件再跑。

---

## 源码来源与板级支持（已逐项核实）

### U-Boot / ATF
仓库 `RuijieNetworksCommunity/bl-mt798x`，分支 **`bl-798x-mod`**（不是 master）。

分支里**已存在**的 BE72 Pro 板级定义：
```
uboot-mtk-20230718-09eda825/configs/mt7988_ruijie_be72_pro_defconfig
uboot-mtk-20230718-09eda825/configs/mt7988_ruijie_be72_pro_multi_layout_defconfig
uboot-mtk-20230718-09eda825/arch/arm/dts/mt7988-ruijie-be72-pro.dts
```

U-Boot DTS 关键内容（说明这确实是给 BE72 Pro 写的）：
```dts
model = "Ruijie RG-BE72 Pro";
blink_led = "sys_green"; system_led = "sys_red";
memory@40000000 { reg = <0 0x40000000 0 0x40000000>; }   // 1GB
mtdparts = "nmbm0:1024k(bl2),512k(u-boot-env),2048k(Factory),2048k(fip),
            512k(product_info),512k(kdump),115200k(ubi)";
reset-button { gpios = <&gpio 13 GPIO_ACTIVE_LOW>; };
mesh-button  { gpios = <&gpio 14 GPIO_ACTIVE_LOW>; };
```

**已发现并绕过的一个坑**：MT7988 平台代码在 `atf-20240117-bacca82a8/plat/mediatek/mt7988/`（58 个文件），
但 BE72 Pro 的 ATF 配置 `mt7988_ruijie_be72_pro_defconfig` 被错误地放在了
**没有 mt7988 平台目录的** `atf-20220606-637ba581b/configs/` 里 —— 也就是说
`SOC=mt7988 BOARD=ruijie_be72_pro ./build.sh` 用仓库默认的 `ATF_DIR` 会直接报 `config not found`。

工作流的 `Patch ATF config for BE72 Pro` 步骤解决它：用同目录下 MTK 自己的
`mt7988_rfb_spim_nand_defconfig` 补齐 `mt7988_ruijie_be72_pro_defconfig`。
两个 ATF 目录的配置键格式还不一样（老版 `CONFIG_*`、新版 `_*`），所以是复制新版的而不是搬旧的。

### 固件
仓库 `RuijieNetworksCommunity/MT798X-6.6-24.10`，分支 **`mt798x-mt799x-mtwifi_be72pro_be68u`**。

`target/linux/mediatek/image/filogic.mk` 里确认有 6 个 BE72 Pro / BE68 Ultra 变体：
```
ruijie_rg-be72-pro            DEVICE_DTS = mt7988d-ruijie-rg-be72-pro
ruijie_rg-be72-pro-mod        VARIANT = (U-Boot mod)
ruijie_rg-be72-pro-dsa-mod    + kmod-rtl8372n_dsa
ruijie_rg-be68-ultra / -mod / -dsa-mod
```

**注意**：社区官方 CI 会从两个**私有**仓库 `rtl837x-gsw-driver` / `rtl837x-dsa-driver`
拉交换芯片驱动（用部署密钥）。本工作流不依赖它们 —— 依据是第三方仓库
`y9858/be72pro` 在**同一分支**上不加这两个仓库也能编译并发布固件。
若将来报缺驱动，把它们的公开 fork 加进 `diy.sh` 即可。

另：`MM798X-6.6-24.10` 的 `feeds.conf.default` 第 6 行是社区私有源，工作流里 `sed -i '6d'` 摘掉了。

### 与上游官方源的关系
**ImmortalWrt 官方和 OpenWrt 官方都没有 BE72 Pro。** 实测：
| 源 | 结果 |
|---|---|
| ImmortalWrt 24.10.4 / 25.12.0-25.12.2 / snapshot filogic | 只有 `ruijie_rg-x60`、`ruijie_rg-x60-pro`、`routerich_be7200` |
| OpenWrt 官方 24.10.4 filogic | 只有 `ruijie_rg-x60-pro` |

`routerich_be7200` **不是**同一台机器（MT7987A + 512MB flash），固件和 FIP 都不通用 —— 别被文件名骗了。

---

## 为什么只刷 FIP、不刷 BL2

设备上引导是分层的：

```
BootROM → BL2 (mtd1, 1MB)  ← 负责 DDR4 初始化、加载 FIP
             ↓
           FIP (mtd6, 2MB) ← BL31 (ATF) + BL33 (U-Boot 本体)
             ↓
           ubi (mtd7)      ← 固件
```

**DDR 训练参数在 BL2 里。** 本方案只替换 FIP、保留原厂 BL2，
所以即使我们编出来的 ATF 里 DDR 相关配置和真机不完全一致，也不会影响启动 ——
DDR 早就被原厂 BL2 初始化好了。这是这套方案风险可控的关键，**不要自作聪明去刷 bl2**。

---

## 风险与前提（读三遍）

1. **刷机不可逆。** 原厂 ReyeeOS 引导被覆盖后，官方固件、Reyee 云管理、Mesh 组网、官方保修全没。
2. **必须先用有线。** U-Boot 恢复模式没有 WiFi。网线接 Mac ↔ 路由器 LAN 口。
3. **必须已有 8 个分区备份**（mtd0~mtd7）。没有就别开始。
4. **云编译产物未经本机验证。** 社区同源码配置的产物被证实可用，但本仓库编出来的这份是新的二进制 —— 刷之前建议先确认能进 U-Boot Web 恢复页（按住 RESET 上电，天线灯变红），进得去就说明 FIP 启动正常。
5. **兜底**：UART 115200 8n1 3.3V + `mtk_uartboot`；最终手段用 mtd0 的 256MB 全片镜像走编程器。

---

## 刷机流程

### Stage 1 — 刷 FIP（SSH）
```bash
# 把 fip.bin 传到路由器 /tmp，然后：
mtd erase  FIP
mtd write  /tmp/fip.bin FIP
mtd verify /tmp/fip.bin FIP      # 必须 Success，否则不要重启
mtd erase  u-boot-env
```

### Stage 2 — U-Boot Web 刷固件
1. 拔电 → 网线接 Mac ↔ 路由器 LAN 口
2. Mac 以太网设静态 IP `192.168.110.2` / 掩码 `255.255.255.0` / 网关留空
3. 按住 RESET 不放 → 插电 → 等约 10 秒 → 天线灯变红 → 松手
4. 浏览器无痕模式开 `http://192.168.110.1`
5. 刷写固件 → 选 sysupgrade.bin → 分区样式按 FIP 变体选（见上表）→ Upload → Update
6. 等自动重启，蓝灯亮 = 好

刷完默认 `192.168.2.1` / `root` / `password`。

工作区根目录的 `flash-be72pro-fip.sh` 把 Stage 1 自动化了（含 6 道安全闸）。

---

## 目录说明

```
.github/workflows/
  build-be72pro-uboot.yml      编 FIP（快，分钟级）
  build-be72pro-firmware.yml   编固件（慢，2-6 小时）
be72pro/
  firmware/mt798x.config       固件 .config（取自 y9858/be72pro 的可用配置）
  firmware/diy.sh              编译前定制脚本（插件/主题/默认IP）
  uboot/                       占位（workflow path 过滤用）
```

## 出处与许可
- `bl-mt798x`、`MT798X-6.6-24.10`：RuijieNetworksCommunity，GPL-2.0
- `mt798x.config` / `diy.sh`：来自 `y9858/be72pro`（个人仓库，自担风险）
- 仅限个人非商业使用
