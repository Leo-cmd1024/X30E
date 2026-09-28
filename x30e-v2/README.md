# X30E 实用向固件（bootstrap + 射频稳定性 + OpenVPN）

针对锐捷 **RG-X30E**（MT7981 / Filogic 820）的 ImmortalWrt 21.02 云编译配方。

- 源码：`RuijieNetworksCommunity/immortalwrt-mt798x` @ `openwrt-21.02`
- 设备 profile：`ruijie_rg-x30e-firmware2`（与在跑固件同 layout，`sysupgrade -k` 可保配置直升）
- Workflow：`.github/workflows/build-x30e-fw.yml`（push 到本目录即自动触发）

---

## 三项改动

### 1. 主题只留 bootstrap

`luci-theme-argon`、`luci-app-argon-config`、`luci-i18n-argon-config-zh-cn`、
`luci-theme-bootstrap-mod` 全部移除，只保留 `luci-theme-bootstrap`。

`files/etc/uci-defaults/zz-x30e-theme` 额外清掉主题注册表里可能残留的
`luci.themes.{Argon,BootstrapDark,BootstrapLight,Bootstrap_Mod}`，并把
`luci.main.mediaurlbase` 锁到 `/luci-static/bootstrap`，下拉框里只剩 Bootstrap。

> `zz-` 前缀是为了在 `/etc/uci-defaults` 的字母序里**最后**执行 ——
> 否则会被 `30_luci-theme-*`、`luci-argon-config`、`99-default-settings*` 覆盖。

### 2. 射频稳定性：可配置的 DFS 策略

无线射频配置页（General Setup，紧跟在「Operating frequency」下面）新增复选框：

**`Allow DFS channels`**

| 状态 | 行为 |
|---|---|
| 勾选（**默认**） | 自动信道选择（ACS）可以选到 DFS 信道 |
| 取消勾选 | ACS **跳过** DFS 信道（中国 5G 为 `52;56;60;64`） |
| 手动固定信道时 | 复选框不影响，手选的信道照旧生效 |

#### 实现原理

不硬编码任何信道，把策略交给驱动自己的 ACS：

1. `files/usr/sbin/x30e-wifi-dfs` 读 `uci wireless.<dev>.dfs`；
2. 从 `/etc/wireless/l1profile.dat` 的 `INDEX0_profile_path` / `INDEX0_main_ifname`
   两个分号列表里按 `phy` 定位当前射频对应的驱动 profile
   （实测：`ra0 → mt7981.dbdc.b0.dat`（2.4G）、`rax0 → mt7981.dbdc.b1.dat`（5G））；
3. 把结果写进该 profile 的 `AutoChannelSkipList`，顺带把 `ACSCheckTime` 置 0。

`AutoChannelSkipList` 是 MTK 驱动**原生**的 ACS 跳过表，`cmm_profile.c` 里这样解析：

```c
pAd->ApCfg.AutoChannelSkipListNum = delimitcnt(tmpbuf, ";") + 1;
for (i = 0, macptr = rstrtok(tmpbuf, ";"); macptr; macptr = rstrtok(NULL, ";"), i++)
        pAd->ApCfg.AutoChannelSkipList[i] = os_str_tol(macptr, 0, 10);
```

即 **分号分隔的十进制信道号，上限 10 条**。运行期由 `AutoChannelSkipListCheck()`
在 ACS 评估信道时逐个比对跳过。参考 MediaTek *Auto-Channel Selection Application Note*。

中国 5G 合法信道为 `36 40 44 48 52 56 60 64 149 153 157 161 165`，
DFS 只有 `52 56 60 64`（100-144 段国内未开放）→ 4 条，远低于上限。

#### 生效链路

```
LuCI 勾/取消复选框
   → uci wireless.MT7981_1_2.dfs
   → wifi reload（netifd）
   → /lib/netifd/wireless/mtwifi.sh  drv_mtwifi_setup()   ← 注入的钩子
        └─ /usr/sbin/x30e-wifi-dfs "$dev"   （改 profile 的 AutoChannelSkipList）
        └─ json_dump | /sbin/mtwifi_cfg setup
             └─ mtwifi_cfg 是「load_profile → 改 → save_profile」，
                所以钩子写进去的值会被原样保留并送进驱动
```

#### 顺带做的两项稳定性调整

- **`ACSCheckTime=0`** —— 关闭周期性自动信道重选。非 0 时驱动每 N 小时重跑一次
  ACS，一旦换信道所有客户端瞬断，是家庭环境里「WiFi 莫名断一下」的常见来源。
  现在只在启动时选一次。
- **`twt=0`** —— 关闭 802.11ax 的 TWT。TWT 是省电特性，但部分手机/IoT 的 TWT
  协商有 bug，表现为间歇卡顿、掉线、时延抖动。

> 这两项都由 `files/etc/uci-defaults/zz-x30e-wifi` 以「缺省才写」的方式设置，
> 不会覆盖用户已经调过的值。

### 3. OpenVPN

`openvpn-openssl` + `luci-app-openvpn` + `luci-i18n-openvpn-zh-cn` + `kmod-tun`。

> 本机 WAN 是运营商 CGNAT（`100.64.x.x`，无公网 IP），**做 OpenVPN 服务端需要先
> 解决入站可达**（端口映射无效）；作为**客户端拨出**可直接使用。若确实要服务端，
> 可考虑 Tailscale / Lucky 的 STUN 穿透打洞。

---

## 补丁机制

`files/` 是 OpenWrt buildroot 的**自定义文件覆盖层**：构建时在包安装**之后**
拷进 rootfs，因此优先级高于包自带文件。本配方用它与覆盖两个文件：

| 覆盖目标 | 来源包 | 改动 |
|---|---|---|
| `lib/netifd/wireless/mtwifi.sh` | mtwifi-cfg | 在 `json_dump \| mtwifi_cfg setup` 前插一行钩子 |
| `www/luci-static/resources/view/network/wireless.js` | luci-mod-network | 在射频 General Setup 里插一个 `form.Flag` |

- **不改任何 feed 源码**，补丁全部集中在 `files/` 下，可审计、可回滚。
- `orig/` 存放覆盖前的原始文件（供 diff 参考），**不会**进固件。
- CI 里有补丁自检 + 语法自检（`sh -n` / `node --check`）+ 构建后 rootfs 深度校验。

> ⚠️ `wireless.js` 的基线取自**设备上的实际文件**（95104 B），不是 feed 的
> `RuijieNetworksCommunity/luci@openwrt-21.02` 版本（2251 行 / 不同）。
> 锐捷 fork 对 `wireless.js` 做过定制（含 `hwtype == 'mtwifi'` 分支）。
> 若后续 feed 该文件有更新，需要用新文件重新 diff 一次，别直接沿用。

---

## 升级

```
scp <本文件>.bin root@192.168.110.1:/tmp/
ssh root@192.168.110.1 "sysupgrade -k /tmp/<本文件>.bin"
```

或在 LuCI：**系统 → 备份/刷写固件** → 勾选「保留配置」→ 上传。

刷完在 **网络 → 无线 → 编辑(5G 射频) → General Setup** 里能看到
`Allow DFS channels` 复选框，默认已勾选。
