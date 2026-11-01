# Skyworth 65G31 / 7S77_G31：Coocaa Android 8 实测笔记

这是一台中国市场的 Skyworth 65G31，系统显示型号为 `7S77_G31`。记录来自一台真实设备，主要用于帮助同型号设备的 Agent 少走弯路；目前还不是经过数日使用验证的可直接复用 debloat list。

## 设备身份

| 项目 | 实测值 |
|---|---|
| 商品型号 | Skyworth 65G31 |
| `ro.product.model` | `7S77_G31` |
| `ro.product.device` | `sky848_9s60` |
| SoC | MStar `m7221`（由 `ro.boot.hardware` 读取） |
| Android / SDK | Android 8.0.0 / API 26 |
| Build | `OPR5.170623.014 dev-keys` |
| ADB | Wi-Fi TCP 5555，`ro.adb.secure=1` |
| 电视系统 | Coocaa OS |

## 开启 ADB

在电视上进入：

```text
设置 → 通用 → 关于本机 → 本机信息 → 更多信息
```

在“更多信息”页面缓慢输入：

```text
↑ ↑ ↓ ↓ ← → ← →
```

进入工厂菜单后，只修改：

```text
通用设置 → ADB 开关 → 开
```

不要修改任何其他选项。另外还需要在设置-应用中，勾选“未知来源安装”。

连接电脑：

```bash
adb connect <tv-ip>:5555
adb devices
```

第一次连接会在电视上显示“允许 USB 调试？”。需要在电视上确认，并建议勾选“始终允许”。电视上也可能并不显示这个提示，只要能正确联通ADB，就不碍事。

## 首次检查和备份

仓库工具需要先安装依赖：

```bash
python3 -m venv tools/venv
tools/venv/bin/pip install -r tools/requirements.txt
```

备份至少包括：

```bash
TV_HOST=<tv-ip> tools/tv.py 'pm list packages -f' > backup/packages-full.txt
TV_HOST=<tv-ip> tools/tv.py 'pm list packages -d' > backup/packages-disabled-before.txt
TV_HOST=<tv-ip> tools/tv.py 'getprop' > backup/getprop.txt
TV_HOST=<tv-ip> tools/tv.py 'settings list system' > backup/settings-system.txt
TV_HOST=<tv-ip> tools/tv.py 'settings list secure' > backup/settings-secure.txt
TV_HOST=<tv-ip> tools/tv.py 'settings list global' > backup/settings-global.txt
```

这台机器的关键初始状态：

- 默认 Home 不回答标准 Android `CATEGORY_HOME`，`cmd package resolve-activity ... HOME` 返回 `No activity found`。
- 原厂 Home 是 `com.tianci.movieplatform/com.coocaa.homepage.vast.HomePageActivity`。
- `com.skyworth.tv` 是 HDMI、AV、电视输入源相关组件，不能禁用。
- 当前固件的 Coocaa Home 逻辑不能按普通 Android TV 的 `set-home-activity` 处理。

## Debloat 范围

本次只使用 `pm disable-user --user 0`，没有卸载应用，也没有刷写分区。

实际禁用的两批包：

### 广告、推送、遥测、自动安装

```text
com.tianci.ad
com.tianci.push
com.coocaa.sysreport
com.baidu.oases
com.tianci.autoinstaller
```

### 不使用的厂商预装

```text
com.coocaa.guide
com.coocaa.educate
com.coocaa.lifeassistant
com.tianci.appstore
com.coocaa.app_browser
com.coocaa.tvmanager
com.coocaa.teampresent
com.coocaa.os.thememanager
com.coocaa.messageprovider
```

执行方式：

```bash
TV_HOST=<tv-ip> tools/disable.sh <list-file>
```

恢复方式：

```bash
TV_HOST=<tv-ip> tools/restore.sh <same-list-file>
```

恢复脚本会读取 `tools/disable.sh` 写入的本地恢复记录，并保留处理前就已经禁用的包状态。

## 必须保留的包和功能

本次选择保留：

- `com.skyworth.tv`：HDMI、AV、电视输入源
- `com.tianci.movieplatform`：原厂 Home，切换失败时的恢复入口
- `com.tianci.system`、`com.coocaa.os.ccosservice`：Coocaa Home、输入源和系统服务
- `com.skyworth.angel.voice`、`com.skyworth.aimanager`：语音助手
- `com.coocaa.miracast`、`com.coocaa.skycast.tool`：Miracast、投屏相关功能
- `com.coocaa.os.upgrade`：系统固件更新
- `com.tianci.user`：Coocaa 账号服务
- `com.tianci.localmedia`：本地媒体和 USB 媒体能力
- Android 核心包、输入法、蓝牙；如果某个同型号固件带有 Play Store/Play Services，也不要禁用它们。

这台设备在处理前已有以下包处于禁用状态：

```text
com.skyworth.smarthome_tv
com.xx.blbl
com.coocaa.iptv
```

`com.xx.blbl` 推测是某个第三方的哔哩哔哩App。电视拒绝了 shell 重新启用它的请求，因此 Launcher 的 Dock 如果找不到它，会按 `fill_dock=true` 用其他已安装应用填充空位。

## Launcher 配置

本次修改的是：

```text
launcher/app/src/main/res/values/config.xml
```

Dock 顺序：

```text
com.xiaodianshi.tv.yst
com.xx.blbl
cn.nascab.android
com.tencent.qqmusictv
com.coocaa.dangbeimarket
```

Coocaa 输入源支持：

```xml
<string name="source_panel_service">
  com.coocaa.os.ccosservice/com.coocaa.os.centerintegration.service.CenterService
</string>
<string name="source_panel_action">
  com.coocaa.intent.action.CONTROL_CENTER
</string>
<string name="source_panel_extra">control_center_command=100</string>
<bool name="coocaa_sources">true</bool>
```

这使 Launcher 的“输入源”按钮可以调用电视自己的 HDMI1、HDMI2、AV 等来源列表，不需要为了 HDMI 位置改 Java 代码。

## 编译和安装

电视是 SDK 26，Launcher 的 `minSdk` 也是 26。构建时使用：

```bash
JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home \
ANDROID_HOME=/Users/<user>/Library/Android/sdk \
./gradlew assembleRelease \
  -PlauncherApplicationId=com.coocaa.tvhome \
  --no-daemon
```

本次 APK 大小为 586,353 bytes，低于项目 589,824 bytes 的限制。

### 重要：安装确认弹窗

本次最初使用 `pm install` 和 install session 时得到：

```text
INSTALL_FAILED_USER_RESTRICTED: Install canceled by user
```

后来确认真实原因是：电视上弹出了安装确认窗口，但没有及时用遥控器确认，超时后系统自动拒绝。不是电视永久禁止 ADB 安装。

正确做法：

1. 开始安装前，先提醒使用者盯着电视。
2. 电脑执行安装命令。
3. 电视出现“允许安装 / 安装确认”后，马上按确定。

```bash
adb connect <tv-ip>:5555
adb install -r launcher/app/build/outputs/apk/release/app-release.apk
```

安装后验证：

```bash
adb shell pm path com.coocaa.tvhome
adb shell am start -W -n \
  com.coocaa.tvhome/com.gathin.tvhome.MainActivity
```

不要在没有确认电视弹窗的情况下连续重试，否则很容易把“确认超时”误判为“固件永久禁止安装”。

## 切换 Home

这台电视不用普通 Android HOME 解析。实际使用两组 Coocaa 属性：

```bash
adb shell '
setprop persist.service.homepage.pkg com.coocaa.tvhome
setprop persist.service.homepage.cls com.gathin.tvhome.MainActivity
setprop persist.service.model.pkg com.coocaa.tvhome
setprop persist.service.model.cls com.gathin.tvhome.MainActivity
'
```

测试注入的 Home 键：

```bash
adb shell input keyevent KEYCODE_HOME
adb shell dumpsys window | grep -E 'mCurrentFocus|mFocusedApp'
```

应看到：

```text
com.coocaa.tvhome/com.gathin.tvhome.MainActivity
```

必须再按一次真实遥控器 Home 键。注入按键通过，不能单独证明实体遥控器路径也通过。

本机观察到 Coocaa 可能在重启过程中重写 `persist.service.model.*`，所以每次改完都要做一次真实重启检查。原厂 Home 没有被禁用，切换失败时仍有恢复入口。

恢复原厂 Home：

```bash
adb shell '
setprop persist.service.homepage.pkg com.tianci.movieplatform
setprop persist.service.homepage.cls com.coocaa.homepage.vast.HomePageActivity
setprop persist.service.model.pkg com.tianci.movieplatform
setprop persist.service.model.cls ""
input keyevent KEYCODE_HOME
'
```

### 真实遥控器 Home 键

主题更新和 Launcher 重启不是电视整机重启，但都可能让 Coocaa 抢回 Home：

- **更换主题**：主题同步会停止并重新启动 Launcher；主题文件本身不会自动修复原厂 Home 属性。
- **重新安装 Launcher**：无障碍服务如果曾经绑定，可能被安装过程清掉。
- **电视整机重启**：Coocaa 可能重写 `persist.service.homepage.*` / `persist.service.model.*`，也可能不绑定第三方无障碍服务。

这台设备没有标准 `com.android.settings`、Accessibility 设置页面，工厂菜单也没有发现无障碍开关。虽然可以把 `HomeKeyService` 写入 `enabled_accessibility_services`，但酷开无障碍管理器没有真正绑定它；`dumpsys accessibility` 仍只显示原厂“便捷面板”。因此不能把 HomeKeyService 当作这台设备的可靠方案。

当前可用的恢复命令是写入两组 Coocaa Home 属性，然后启动 Launcher：

```bash
adb shell '
setprop persist.service.homepage.pkg com.coocaa.tvhome
setprop persist.service.homepage.cls com.gathin.tvhome.MainActivity
setprop persist.service.model.pkg com.coocaa.tvhome
setprop persist.service.model.cls com.gathin.tvhome.MainActivity
am start -n com.coocaa.tvhome/com.gathin.tvhome.MainActivity
'
```

每次换主题、重新安装 Launcher 或整机重启后，都应按一次真实遥控器 Home 键验证。只用 `input keyevent KEYCODE_HOME` 不足以证明实体遥控器路径正常。

如果主题工具需要适配这台电视，主题同步脚本应在重启 Launcher 后重新写入上面的两组属性；直接使用仓库默认的 `tools/theme.sh` 还会遇到包名差异，因为默认脚本写的是 `com.gathin.tvhome`，本机安装包是 `com.coocaa.tvhome`。

## 开机广告

单独禁用 `com.tianci.ad` 不足以消除开机广告。广告还涉及 MStar/Coocaa 的 boot video 机制；同系列设备记录显示，开机视频由 `com.skyworth.tv` 播放，相关文件属于系统侧资源。

在这台 65G31 上，最终由使用者确认有效的设置是：

```bash
adb shell '
setprop persist.sys.bootvideo.time 0
setprop persist.sys.ad.startboot ""
'
```

然后重启验证。`persist.sys.bootvideo.time=1` 在其他同系列设备上有不同记录，但在这台 G31 上曾改变广告关闭提示，不应直接套用。

## ADB 安全收尾

如果不再继续调试，关闭网络 ADB：

```bash
adb shell '
setprop persist.sys.openadb 0
setprop persist.service.adb.enable 0
setprop persist.adb.tcp.port -1
setprop service.adb.tcp.port -1
settings put global adb_enabled 0
'
```

再用仓库工具验证：

```bash
tools/venv/bin/python tools/adb-closed.py <tv-ip> 5555
```

只有电视主动返回 `Connection refused`，才算端口确实关闭。

## 本次测试的边界

- 已实测安装、Launcher 启动、Coocaa 输入源配置和广告设置。
- 已实测两批禁用包，未做数日稳定性观察。
- 原厂 Home 没有禁用，因此恢复风险较低。
- 哔哩哔哩处理前已禁用，未能通过 shell 重新启用。
- Home 属性在不同重启时可能被 Coocaa 重写，必须测试真实遥控器和冷启动。
- 不要把这台设备的包清单直接套到其他 Skyworth/Coocaa 型号；同品牌不同 `ro.product.device` 的关键包和启动逻辑可能不同。
