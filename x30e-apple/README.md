# X30E Apple-UI 固件（云端编译）

给锐捷 **RG-X30E**（MT7981 / ImmortalWrt 21.02-SNAPSHOT）做一版 **苹果风格** LuCI。

产出物：`immortalwrt-mediatek-mt7981-ruijie_rg-x30e-firmware2-squashfs-sysupgrade.bin`
（与设备当前 profile 一致 → **可保留配置直接升级**）

---

## 1. 它改了什么

**只动 UI 层，不碰内核 / 网络 / 驱动。**

| 层 | 做法 |
|---|---|
| 样式 | 编译前把 `apple.css` 追加到 `luci-theme-argon/.../css/cascade.css` 末尾；`apple-dark.css` 追加到 `dark.css` |
| 主题参数 | `files/etc/uci-defaults/99-apple-ui` → Argon 主色 `#007AFF`、磨砂 20px、跟随系统亮暗、关掉在线壁纸 |
| 附加包 | `luci-app-argon-config`（UI 里可再微调） |

**为什么不新建主题**：重建一套 LuCI 主题要复刻全部模板，回归风险高。
追加覆盖靠 CSS 源码顺序取胜，**零模板侵入**，任何一处不满意都能用一条 uci 命令回到 bootstrap。

## 2. 苹果风格具体落在哪

| 元素 | 处理 |
|---|---|
| 字体 | `-apple-system` / SF Pro → 苹方 PingFang SC → 雅黑。**在 Mac/iPhone 上看就是真·系统字体**，不需要打包字体文件 |
| 配色 | iOS 系统色：蓝 `#007AFF`、绿 `#34C759`、红 `#FF3B30`、橙 `#FF9500`；暗色 `#0A84FF` 等 |
| 顶栏 / 侧栏 | 半透明 + `backdrop-filter: saturate(180%) blur(20px)` + 1px 发丝线（macOS 窗口观感） |
| 卡片 | 白底、16px 圆角、极淡描边 + 柔和投影（iOS 分组列表） |
| 表单 | 标签左对齐、输入框无边框填充式、聚焦时白底 + 蓝色 4px 焦点环 |
| 下拉 | 原生 select 换成 SF 风格箭头 SVG；下拉浮层 12px 圆角 + 大投影 |
| 按钮 | 蓝底实心 = 提交类；红调 = 删除/重置；灰底蓝字 = 次要操作 |
| 标签页 | iOS 分段控件（灰底轨道 + 白色滑块 + 投影） |
| 表格 | 去掉斑马纹，改成 iOS 分隔线 + 悬停淡蓝 |
| 提示条 | 圆角 14px，按 notice/warning/error/success 上 iOS 语义色 |
| 滚动条 | 细条 + 半透明拇指，无轨道 |
| 登录页 | 居中磨砂卡片、全宽主按钮 |
| 亮/暗 | 两套完整实现，默认跟随系统（`mode=normal`） |

**安全边界**：只改颜色 / 圆角 / 阴影 / 边框 / 字体 / 对齐，
**不碰** `display` `position` `float` `width` `height`，所以不会把布局搞崩。

## 3. 编译

push 到 `x30e-apple/**` 或本工作流文件即自动触发；也可在 Actions 里手动 `Run workflow`。

- 源码：`RuijieNetworksCommunity/immortalwrt-mt798x` @ `openwrt-21.02`
- 基线配置：`defconfig/mt7981-ax3000.config`，只启用 `ruijie_rg-x30e-firmware2`
- 耗时：约 1.5–3 小时（首次要编工具链）
- 产出：GitHub Release `x30e-appleui-<run>`

## 4. 保留配置升级

```bash
# 1. Mac 上下载 release 里的 sysupgrade.bin，传到路由器
scp -P 22 <文件>.bin root@192.168.110.1:/tmp/
# 2. 路由器上执行（-k = 保留配置）
ssh -p 22 root@192.168.110.1 "sysupgrade -k /tmp/<文件>.bin"
```

LuCI 方式：**系统 → 备份/刷写固件 → 勾选「保留配置」→ 上传**。

刷完强刷页面即可；`/etc/config` 原样保留（PPPoE、WiFi、防火墙、DHCP 都不丢）。

## 5. 回退

**只回退外观**（不用重刷固件）：

```bash
uci set luci.main.mediaurlbase='/luci-static/bootstrap'
uci commit luci && /etc/init.d/uhttpd restart
```

**回退整套固件**：刷回上一版 sysupgrade（同样 `-k`）。

## 6. 微调

装了 `luci-app-argon-config`，在 **系统 → Argon 主题** 里可调主色、模糊半径、透明度、亮/暗模式。
注意：调主色会覆盖 `99-apple-ui` 的 `#007AFF`（这属于预期行为 —— 那是配置层，不是 CSS 层）。

## 7. 目录

```
apple.css                 ← 追加到 cascade.css（亮色 + 通用）
apple-dark.css            ← 追加到 dark.css（暗色）
files/
  etc/uci-defaults/99-apple-ui   ← 首次启动/升级后自动应用主题参数，跑完自删
build-x30e-appleui.yml    ← 工作流（推送到 .github/workflows/）
```
