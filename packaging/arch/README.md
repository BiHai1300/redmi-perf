# Arch Linux 打包

这里提供 [AUR](https://aur.archlinux.org/) 风格的 PKGBUILD，把 `redmi-perf`
拆成两个包：

| 包 | 内容 |
|---|---|
| `redmi-perf-git` | `/usr/bin/redmi-perf` 命令行工具 |
| `redmi-perf-dkms-git` | `redmi_perf_wmi` 内核模块源码（DKMS） |

`redmi-perf-git` 依赖 `redmi-perf-dkms-git`，装 CLI 会一起装上模块。

> **不打包 `nvpcf-shim`**：它要替换 / 重编 NVIDIA 内核模块，属于高风险、
> 与具体驱动版本强绑定的操作，不适合交给包管理器。需要时按
> [../../docs/arch-linux.md](../../docs/arch-linux.md) 里的步骤手动安装。

## 构建与安装

```bash
sudo pacman -S --needed base-devel dkms cargo git
cd packaging/arch/redmi-perf-git
makepkg -si
```

`dkms` 的 pacman 钩子会在安装后自动为已装内核编译模块
（安装 `/usr/src/redmi-perf-<pkgver>/dkms.conf` 触发）。
内核头文件由各内核包提供，`dkms` 已把它列为可选依赖，本包不重复依赖。

安装完成后：

```bash
sudo modprobe redmi_perf_wmi     # 立即加载
redmi-perf get                   # 看当前性能模式
```

开机自动加载（可选，包不会替你做，避免强制加载内核模块）：

```bash
echo redmi_perf_wmi | sudo tee /etc/modules-load.d/redmi-perf.conf
```

## 版本规则

这是 **VCS 包**（`-git` 后缀），`pkgver` 由 `pkgver()` 从 git 生成：

- 有 tag 时：`0.2.0.r3.gabcdef0`（tag + 提交数 + 哈希）
- 无 tag 时：`r3.8941a19`（提交数 + 哈希）

模块的 `PACKAGE_VERSION` 和 `/usr/src/redmi-perf-<pkgver>` 目录名都用
PKGBUILD 的 `$pkgver` 生成，符合 [DKMS 打包指南](https://manual.archlinux.page/package-guidelines/dkms/)。

上游目前还没有 release tag，所以只提供 `-git` 包。等 `v0.2.0` 之类的
tag 发布后，可以再加一个稳定版 PKGBUILD：从
`$url/archive/v$pkgver.tar.gz` 取源码、用 `updpkgsums` 生成校验和，
`pkgver` 直接用 tag 里的版本号。

## 本地测试（还没 push 的改动）

`-git` 包默认从上游仓库拉取：

```
_repo="https://github.com/wanxinhao/redmi-perf.git"
```

想用自己 fork 或本地工作区测试，改 `_repo` 再 `makepkg -si`：

```bash
# fork
sed -i 's|wanxinhao/redmi-perf|BiHai1300/redmi-perf|' PKGBUILD

# 本地仓库（包含未 push 的 commit）
sed -i 's|https://github.com/wanxinhao/redmi-perf.git|file:///path/to/redmi-perf|' PKGBUILD
```

## 从脚本安装迁移

如果你之前跑过 `scripts/install-redmi-perf.sh --dkms`，DKMS 里会有一个
`redmi-perf/0.2.0`。先清掉，否则内核升级时两个版本会重复编译、抢同一个模块名：

```bash
sudo ./scripts/install-redmi-perf.sh --uninstall
```

然后再 `makepkg -si`。

## 卸载

```bash
sudo pacman -Rns redmi-perf-git redmi-perf-dkms-git
```

DKMS 钩子会自动 `dkms remove`；`.install` 会提示你 `modprobe -r
redmi_perf_wmi`、以及删掉可选的 `/etc/modules-load.d/redmi-perf.conf`。
