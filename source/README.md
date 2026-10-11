# source/ —— 上游源码快照

```
libfprint/   上游 libfprint 在提交 ba10c9398fe4542ff6403549884d0c8687182845 的完整工作树
             （2026-07-15，"Add fpcmoh driver for FPC Disum match-on-host sensors"）
```

- 351 个文件，约 28 MB
- 已去除 `.git` 元数据，**文件内容未作任何修改**
- 许可：LGPL-2.1-or-later（见 `libfprint/COPYING`）

## 为什么把源码放进仓库

该驱动所在的 merge request
[!570](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/570)
截至收录时**仍未合并**，其分支随时可能变动甚至消失。把"确实能用"的那个提交
完整存档下来，可以保证这份代码不会因上游变动而失传，也方便离线查阅与审计。

## 构建时仍以上游为准

`PKGBUILD` 里的 `source=` 依旧从上游 GitLab 拉取同一个提交：

```
libfprint::git+https://gitlab.freedesktop.org/libfprint/libfprint.git#commit=ba10c939...
```

这样产物与 AUR 包 `libfprint-fpc1022` 保持一致。本快照用于**离线查阅**与
**自行构建验证**，具体步骤见根目录 [README](../README.md)。

## 如何核对这个快照

```bash
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git /tmp/lf-check
cd /tmp/lf-check
git fetch origin ba10c9398fe4542ff6403549884d0c8687182845
git checkout FETCH_HEAD
# 与仓库内 source/libfprint/ 比较，应无差异：
diff -r --exclude=.git /tmp/lf-check /path/to/repo/source/libfprint && echo "一致"
```
