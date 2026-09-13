# Redmi Book 14 内置指纹启用记录（FPC Disum 10a5:9201）

生成时间：2026-09-13 · 机型：Redmi Book 14 · 系统：CachyOS (Arch 系) · 内核 7.2.0

## 结论

指纹已可用：

- **录入**：右手食指（`right-index-finger`）
- **校验**：`fprintd-verify` → `verify-match`
- **sudo 提权**：已实测成功（journal 记录 `lweih : TTY=pts/2 ... COMMAND=/usr/bin/id` + `session opened for user root`）

## 为什么需要额外折腾

你的指纹是 **FPC Sensor Controller `10a5:9201`**（FPC Disum 系列，match-on-host 架构）。

- 上游 **libfprint 1.94.100 完全不支持它** —— 所有 FPC 驱动（`fpcmoc` 等）都不含 `0x9201`，它只躺在 udev 的"已知但不支持"名单里。装官方包只会得到 `No devices available`。
- 唯一支持它的驱动是**尚未合并**的 libfprint MR
  [!570](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/570)（状态 `opened`）新增的 `fpcmoh` 驱动。
- 该 MR 只声明支持 `10a5:9200` 与 `10a5:9201`，但**作者只在 9200 上测过**。你的是 9201 —— 本机实测**可用**（见下）。

实测握手日志（本机固件 `21.26.2.50`）：

```
FPC firmware version: 21.26.2.50
TLS key HMAC verified OK
PSK decrypted: 32 bytes
TLS handshake completed! cipher=PSK-AES128-CBC-SHA256 version=TLSv1.2
```

即 9201 与该驱动期望的 FPC Disum 协议完全兼容。

## 系统上到底改了什么

1. **新增包** `libfprint-fpc1022 1.94.10.r2.gba10c93-1`
   - 构建自 MR !570 的提交 `ba10c9398fe4542ff6403549884d0c8687182845`（2026-07-15）
   - `conflicts=libfprint`、`provides=libfprint=1.94.10`、`provides=libfprint-2.so=2-64`
   - 原 `libfprint 1.94.100-1.1` 已被移除
2. **新增依赖** `fprintd 1.94.5-2.1`、`opencv 5.0.0`（`fpcmoh` 用 SIGFM 匹配器，SIGFM 需要 OpenCV，**这是硬依赖**）
3. **GStreamer 组同步升级** `1.28.6-2 → 1.28.7-1.1`（因为 opencv 依赖 gst，而旧版 gst-libav 锁死了旧 gstreamer，属于被动连带升级）
4. **PAM**：`/etc/pam.d/sudo` 首行插入（**只影响 sudo，登录界面未改**）

```
auth		sufficient	pam_fprintd.so		timeout=15 max-tries=1
```

备份：`/etc/pam.d/sudo.bak.20260913-182801`
另有 snapper 事务快照（`root: 60` ~ `root: 64`）可回滚。

> 注：本机 `libfprint-fpc1022` 包不含 3 个内部 API 的 gtk-doc HTML（MR 改了私有头文件所致），纯开发者文档，不影响任何功能。

## 日常使用

- 跑 `sudo` 时先出现 `请把您的右手食指放在指纹读取器上`，**15 秒内按一下电源键指纹区即通过**；没按或没匹配上会自动回落到原来的密码提示（**密码始终可用**）。
- 指纹模板存在 `/var/lib/fprint/lweih/`，重启后保留。
- `fprintd` 是 D-Bus 激活服务，重启后自动可用，无需 enable。

### 觉得等 15 秒烦？

编辑 `/etc/pam.d/sudo`，把 `timeout=15` 改小（例如 `timeout=5`），或者整行删掉即恢复纯密码。

其他可用选项（详见 `man pam_fprintd`）：`max-tries=N`（默认 3，我们设了 1，即一次不中就转密码）。

想**连 SDDM 登录界面也用指纹**，则在 `/etc/pam.d/system-auth` 首行加同样一行 —— 但那会影响所有登录路径，改前务必保留好备份和可用的 root 通道。

## 回滚（三级，按需选择）

**第 1 级：只取消 sudo 指纹**
```bash
sudo cp /etc/pam.d/sudo.bak.20260913-182801 /etc/pam.d/sudo
```

**第 2 级：整体退回官方 libfprint**（指纹功能消失，系统恢复原样）
```bash
sudo pacman -Rdd libfprint-fpc1022
sudo pacman -S libfprint        # 重新装官方版
sudo systemctl restart fprintd
```

**第 3 级：连依赖一起还原**
在第 2 级基础上，如不再需要则移除 `fprintd`、`opencv`，并可把 GStreamer 组降回（可选，通常没必要）：
```bash
sudo pacman -Rns fprintd opencv
```

## 重装 / 日后维护

安装包与 PKGBUILD 已持久化：

- `/home/lweih/aur/libfprint-fpc1022/libfprint-fpc1022-1.94.10.r2.gba10c93-1-x86_64.pkg.tar.zst`
- `/home/lweih/aur/libfprint-fpc1022/PKGBUILD`
- 另外副本在 `/var/cache/pacman/pkg/`

重装：
```bash
sudo pacman -Rdd libfprint && sudo pacman -U /home/lweih/aur/libfprint-fpc1022/libfprint-fpc1022-*.pkg.tar.zst
```

重建（例如 MR 更新后）：
```bash
cd /home/lweih/aur/libfprint-fpc1022 && makepkg -d --nocheck -f
```

AUR 上也有同名包可跟踪后续更新：`yay -S libfprint-fpc1022`
（注意：本机包用 `-Dudev_hwdb=enabled`，上游默认在 udev ≥ 248 时**跳过**安装
`60-autosuspend-libfprint-2.hwdb`；而 systemd 自带的 hwdb 里**没有任何 10A5 条目**，
所以我们必须装上它来保住设备的 `ID_AUTOSUSPEND=1 / ID_PERSIST=0`。）

**若某次系统升级后发现指纹失效**，先查：
```bash
pacman -Q libfprint libfprint-fpc1022   # 若 libfprint 又被装回来，就是它覆盖了
```
用上面的"重装"命令恢复即可。

## 风险与注意事项

- 驱动来自**未合并的 MR**，属于实验性支持；`10a5:9201` 连作者都没测过（你这里能跑是实测结果，不代表 FPC 官方保证）。
- 官方研究的可靠性数据（在 9200 上）只有 22/30 的真人匹配率，且对指纹**摆放位置敏感**（居中 6/6，旋转后 1/5）。**不要把密码通道关掉**——当前配置始终保留密码。
- 本机 `lweih` 在 `empower` 组，`pkexec` 可免密取得 root。这既是方便也是安全面，请知悉。


---

## 追加（2026-09-13 晚）：polkit 也接入指纹

- **新建 `/etc/pam.d/polkit-1`**（此前不存在，原走 vendor 默认 `/usr/lib/pam.d/polkit-1`）：

```
auth       sufficient   pam_fprintd.so   timeout=15 max-tries=1
auth       include      system-auth
account    include      system-auth
password   include      system-auth
session    include      system-auth
```

- **实测通过**：polkit 弹窗会显示「请把手指放在识别器上」，按一下即授权（约 2 秒）。
  日志证据：`polkit-agent-helper@18` 成功、`fprintd` 被 D-Bus 唤醒被调用，
  且**没有** `pam_unix(polkit-1:auth)` —— 认证阶段由 pam_fprintd 满足。
- 作用域：只影响 polkit。SDDM 登录、sudo 均不受影响（sudo 另有 `/etc/pam.d/sudo`）。
- 注意：`org.freedesktop.policykit.exec` 默认要求是 `auth_admin`（**无缓存**），
  每次提权都会问一次；指纹失败/超时会自动回落密码提示。
- **回滚**：`sudo rm /etc/pam.d/polkit-1` 即恢复 vendor 默认行为。
- 安全性权衡：polkit 授权 ≈ root 级操作，而生物识别不可撤销（指纹无法更换）。
  密码回退始终保留；若想削弱指纹权重，把 `timeout=15` 调小（如 5 秒）即可。
