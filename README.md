# libfprint-fpc1022

> **English summary** — Out-of-tree [libfprint](https://gitlab.freedesktop.org/libfprint/libfprint)
> packaging for **FPC Disum match-on-host** fingerprint readers (`10a5:9200` / `10a5:9201`),
> built on the **unmerged** upstream merge request
> [!570](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/570).
> The repo contains the full pinned source snapshot, an Arch PKGBUILD, a read-only
> diagnostic script, and field notes from getting it working on a Redmi Book 14
> (`10a5:9201`, firmware `21.26.2.50`).

FPC Disum（match-on-host）指纹传感器在 Linux 上的**非官方**支持：Arch 打包 + 完整源码快照 + 实测记录。

## 这是什么

`10a5:9200` / `10a5:9201` 这两个 FPC 指纹传感器**上游 libfprint 至今不支持**（驱动只存在于一个
**尚未合并**的 merge request 里）。本仓库提供：

1. **`source/libfprint/`** —— 上游 MR !570 钉定提交的**完整源码快照**（存档 / 离线查阅 / 自行构建验证）
2. **`PKGBUILD`** —— Arch 打包配方，可直接产出 `libfprint-fpc1022` 包
3. **`fp-status`** —— 只读状态自检脚本（设备、已录指纹、sudo 配置、实时校验）
4. **`fingerprint-fpcmoh-说明.md`** —— 中文实操记录：安装、调优、三级回滚、故障对照

## 实测状态

| 项目 | 值 |
|---|---|
| 设备 | FPC Disum match-on-host（USB `10a5:9200` / `10a5:9201`） |
| 实测机型 | Redmi Book 14（`10a5:9201`），固件 `21.26.2.50` |
| 系统 | CachyOS（Arch 系），内核 7.2.x |
| 结果 | ✅ 录入 / 校验 / **sudo 指纹提权** / **polkit 指纹** 全部实测通过 |

硬件握手证据（本机）：

```
FPC firmware version: 21.26.2.50
TLS key HMAC verified OK
PSK decrypted: 32 bytes
TLS handshake completed! cipher=PSK-AES128-CBC-SHA256 version=TLSv1.2
```

## 源码来源

驱动来自**尚未合并**的上游提交：

- Merge request: <https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/570>
- 钉定提交: `ba10c9398fe4542ff6403549884d0c8687182845`
  （2026-07-15，*"Add fpcmoh driver for FPC Disum match-on-host sensors"*）

`source/libfprint/` 即该提交的完整工作树（351 个文件，28 MB，已去除 `.git`）。

> **构建时 `PKGBUILD` 仍从上游 GitLab 拉取同一个提交**，以保证产物与 AUR 包一致；
> 仓库内的快照用于存档与离线验证（见下文「离线构建」）。

## 目录结构

```
README.md                    本文件
LICENSE                      本项目自有内容的许可（MIT）
NOTICE                       第三方代码/素材的来源与许可说明
.gitignore
PKGBUILD                     → libfprint-fpc1022/PKGBUILD
fp-status                    指纹状态自检脚本（只读，可直接运行）
fingerprint-fpcmoh-说明.md    中文实操记录（安装 / 调优 / 回滚 / 排错）
source/
  README.md                  源码快照的来源与校验说明
  libfprint/                 上游源码快照（LGPL-2.1-or-later）
```

## 安装（Arch / 衍生版）

**方式一：AUR**

```bash
yay -S libfprint-fpc1022
```

**方式二：本仓库构建**

```bash
# 依赖（opencv 是硬依赖，见下）
sudo pacman -S --needed base-devel git opencv meson ninja gobject-introspection gtk-doc

cd libfprint-fpc1022
makepkg -d --nocheck -f

# 该包 conflicts=libfprint，需要先移除官方包；pacman -U 不会自动确认这一步
sudo pacman -Rdd libfprint
sudo pacman -U libfprint-fpc1022-*.pkg.tar.zst

sudo systemctl restart fprintd
fprintd-enroll -f right-index-finger     # 录指纹
fp-status -v                             # 验证
```

**方式三：预编译包** —— 见本仓库 **Releases** 页面（`libfprint-fpc1022-*.pkg.tar.zst` 作为附件提供，
不放进 git 历史）。

## ⚠️ 硬依赖：OpenCV

`fpcmoh` 使用 libfprint 的 **SIGFM** 匹配器（SIFT 关键点匹配），因此 **OpenCV 是编译期硬依赖**。
若 meson 找不到 `opencv4` / `opencv5`，该驱动**不会被编进库里**，装了也不会认设备。

验证驱动是否被编译进去：

```bash
meson setup build -Ddrivers=default -Ddoc=false -Dinstalled-tests=false   # 配置输出里应含 fpcmoh
# 或安装后：
fprint-list-supported-devices | grep 10a5        # 应出现 10a5:9200 / 10a5:9201
```

## 离线构建（直接用仓库内源码）

不经过 PKGBUILD，直接构建验证快照本身：

```bash
cd source/libfprint
meson setup build -Ddrivers=default -Ddoc=false -Dinstalled-tests=false --prefix=/usr --buildtype=release
meson compile -C build
# 不安装的话，可用构建出的示例程序直接测试：
LD_LIBRARY_PATH=build/libfprint GI_TYPELIB_PATH=build/libfprint python3 -c "
import gi; gi.require_version('FPrint','2.0')
from gi.repository import FPrint
c=FPrint.Context(); c.enumerate(); print([d.get_driver() for d in c.get_devices()])"
```

## 风险与免责声明（请务必阅读）

- 该驱动来自**未合并的 MR**，属**实验性**支持，可能随上游变动而失效。
- 上游作者**只在 `10a5:9200` 上测试过**；`10a5:9201`（本项目的实测机型）**未经作者验证** ——
  本仓库的记录是该型号可用的实测证据，但不构成任何保证。
- 第三方可靠性数据表明此类传感器**对按压位置敏感**（居中 6/6，旋转后 1/5）。
- **不要把指纹作为唯一认证方式**，务必保留可用的密码通道与 root 救援手段。
- 本项目与 libfprint 上游、FPC（Fingerprint Cards）、任何硬件厂商**均无关联**，非官方项目。
- 使用前请自行评估风险；因使用本仓库内容导致的任何损失，作者不承担责任。

## 许可

| 内容 | 许可 |
|---|---|
| 本项目自有内容（README、`fp-status`、说明文档等） | **MIT**（见 [LICENSE](LICENSE)） |
| `source/libfprint/` | **LGPL-2.1-or-later**（见 [source/libfprint/COPYING](source/libfprint/COPYING)） |
| `PKGBUILD` | 源自 AUR 包 [`libfprint-fpc1022`](https://aur.archlinux.org/packages/libfprint-fpc1022)（作者 buffmio），沿用其许可 |

详情见 [NOTICE](NOTICE)。
