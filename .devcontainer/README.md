# Dev Container for luci-app-adguardhome（IntelliJ IDEA）

在 Docker 容器里为 `luci-app-adguardhome` 提供**可编辑 + 可编译 ipk/apk** 的开发环境，
编译流程与 `.github/workflows/release_all_versions.yml` 一致，无需在宿主机装任何工具链。

| 文件 | 作用 |
| --- | --- |
| `devcontainer.json` | 容器定义：构建本目录 Dockerfile、以 `openwrt` 用户运行、JetBrains 后端为 IntelliJ、把 SDK 缓存放进持久化卷 `luci-agh-sdk-cache` |
| `Dockerfile` | `ubuntu:24.04`（强制 `linux/amd64`）+ OpenWrt 官方编译宿主依赖 + lua/shell 检查工具 |

## 前置条件

- **IntelliJ IDEA Ultimate**（2025.1+ 才有 IDE 内 Dev Containers 入口）
- **Docker**：官方支持 Docker Desktop。Mac 上用 OrbStack 多数情况可用；
  若 IDEA 里创建 Dev Container 报 Docker 相关错误，临时切 Docker Desktop 验证
  （`docker context use desktop-linux`）。
- Git ≥ 2.20.1，SSH agent 已运行（JetBrains 需要）。
- CPU：镜像固定 **amd64**（OpenWrt SDK 宿主工具是 x86_64 二进制），
  Apple Silicon 上经 Rosetta/x86 模拟运行。纯数据包编译很快，
  但**首次编译会先构建 OpenWrt 基础依赖链（kernel、kmod、固件等），在模拟器下较慢，请用并行 make（见下）**。

## 第一次使用（IDEA 内操作）

1. 用 IDEA 打开本仓库（根目录含 `.devcontainer/devcontainer.json`）。
2. 确认 Docker 在运行（Services 工具窗 → Docker）。
3. 打开 `.devcontainer/devcontainer.json`，编辑器左侧 Dev Container 图标（☁️/🐳）→
   **Create Dev Container and Mount Sources…**。
4. 等待镜像构建与容器创建（首次拉 ubuntu + 装依赖约几分钟），完成后点 **Connect**，
   项目在 JetBrains Client 中打开。容器终端默认工作目录即项目根目录
   （JetBrains 挂载路径为 `/IdeaProjects/luci-app-adguardhome`，下面命令用 `$PWD` 兼容）。

> `openwrt` 用户 UID=501，与你 macOS 主用户一致，容器内可直接读写挂载的项目文件；
> Linux 宿主要把 Dockerfile 里 `-u 501` 改成 `id -u` 的输出。

## 安装 OpenWrt SDK（一次性，首次约 10–30 分钟）

以下用默认目标 **OpenWrt 24.10.6 / x86-64** 举例；换版本/平台时到
`https://downloads.openwrt.org/releases/<版本>/targets/<架构>/<子架构>/` 找 `openwrt-sdk-*Linux-x86_64.tar.{xz,zst}` 的真实文件名替换。

```bash
# 1) 下载并解压 SDK（注意：.tar.zst 需要 --use-compress-program=zstd）
SDK_VER=24.10.6
SDK_DIR="$HOME/.openwrt/sdk/$SDK_VER"          # 落在 docker volume 上，重建容器不丢
SDK_URL="https://downloads.openwrt.org/releases/$SDK_VER/targets/x86/64/openwrt-sdk-24.10.6-x86-64_gcc-13.3.0_musl.Linux-x86_64.tar.zst"
mkdir -p "$HOME/.openwrt/dl" "$SDK_DIR"
curl -fSL -o "$HOME/.openwrt/dl/sdk.tar" "$SDK_URL"
tar --use-compress-program=zstd -xf "$HOME/.openwrt/dl/sdk.tar" -C "$SDK_DIR" --strip-components=1

# 2) 拉取 feeds（base/packages/luci/routing/telephony）
cd "$SDK_DIR"
./scripts/feeds update -a
./scripts/feeds install -a

# 3) 把本项目放进去并生成配置（每次编译前重复执行即可）
mkdir -p "$SDK_DIR/package"
rsync -a --delete --exclude .git "$PWD/luci-app-adguardhome/" "$SDK_DIR/package/luci-app-adguardhome/"
make defconfig

# 4) 关闭 BUILDBOT（推荐，本机 SDK 已生效）
#    SDK 默认 BUILDBOT=y：每轮编译后会清空各包构建目录，并默认勾选全部 kmod。
#    关掉后中间产物保留、重复编译不再触发重建。
perl -0pi -e 's/(config BUILDBOT\n\tbool\n\tdefault )y/${1}n/' Config-build.in
sed -i 's/^CONFIG_BUILDBOT=y/# CONFIG_BUILDBOT is not set/' .config
make defconfig
grep -E '^#? ?CONFIG_BUILDBOT' .config     # 期望：# CONFIG_BUILDBOT is not set
```

- feeds 走 `git.openwrt.org` 太慢/失败时：编辑 `$SDK_DIR/feeds.conf`，把域名换成 GitHub 镜像
  （`git.openwrt.org/openwrt/openwrt.git` → `github.com/openwrt/openwrt.git`，
  `/feed/packages.git` → `/openwrt/packages.git`、`/project/luci.git` → `/openwrt/luci.git`，routing/telephony 同理），再重跑 `./scripts/feeds update -a`。

## 编译（出 ipk）

```bash
SDK_DIR="$HOME/.openwrt/sdk/24.10.6"
# 同步最新源码（含你刚编辑的文件）
rsync -a --delete --exclude .git "$PWD/luci-app-adguardhome/" "$SDK_DIR/package/luci-app-adguardhome/"

cd "$SDK_DIR"
# 生成译文 .lmo 需要 luci-base 的宿主工具 po2lmo（首次执行一次，失败可忽略则译文缺失）
make package/feeds/luci/luci-base/host/compile V=s >/dev/null 2>&1 || true

# 并行编译（串行在 Apple Silicon 模拟下可能耗时数小时）
make -j"$(nproc)" package/luci-app-adguardhome/compile
```

产物：`bin/packages/x86_64/base/luci-app-adguardhome_1.8-r13_all.ipk`（含 zh-cn/en 的 `.lmo` 译文）。
取回宿主：容器终端里 `cp bin/packages/x86_64/base/*.ipk "$PWD/"` 即可出现在项目目录。

**实测耗时（M2 Max + OrbStack 模拟 x86，BUILDBOT 已关闭）：**

| 场景 | 耗时 |
| --- | --- |
| 首次完整构建（含 kernel/kmod/固件下载 + 依赖编译，`-j10`）| ~14 分钟 |
| 之后每次改动源码后重新编译 | ~3.5 分钟 |

其中 3.5 分钟基本是**模拟环境下遍历 28 个依赖目标的固定开销**（每个约 7s），
依赖不会被重编（实测 `clean-build` 次数为 0、无 `.built` 刷新）；真正的打包阶段是秒级。
产物可复现：同一份源码两次编译出的 ipk sha256 完全一致（`SourceDateEpoch` 固定）。

OpenWrt 25.x SDK 产物是 apk（路径下 `luci-app-adguardhome-*.apk`）；版本矩阵见 CI workflow。

## 编译 AdGuardHome 核心包（可选）

`AdGuardHome/` 是本仓库另一个包（AGH 核心，Go + 前端）。它**不在 CI 里构建**，
且自 v0.107 起上游改为「单二进制 + `go:embed` 前端」，因此需要额外的工具链：
**Go ≥ 1.26**（go.mod 要求）与 **Node 20**（`client_v2` 前端），OpenWrt 24.10 的
`golang/host`(1.23) / `node/host` 都不适用。

**一次性准备工具链**（已装在本机卷内，此处供新环境参考）：

```bash
mkdir -p ~/.openwrt/tools && cd ~/.openwrt/tools
# Go：AdGuardHome 的 go.mod 要求 1.26.6
curl -fSL -o go.tgz https://go.dev/dl/go1.26.6.linux-amd64.tar.gz && tar xzf go.tgz && rm go.tgz
# Node 20（上游 CI 使用的版本）
F=$(curl -fsSL https://nodejs.org/dist/latest-v20.x/ | grep -oE 'node-v20[0-9.]+-linux-x64\.tar\.xz' | head -1)
curl -fSL -o node.tar.xz "https://nodejs.org/dist/latest-v20.x/$F" && tar xf node.tar.xz \
  && mv "${F%.tar.xz}" node && rm node.tar.xz
~/.openwrt/tools/go/bin/go version && ~/.openwrt/tools/node/bin/node -v
```

**编译**：

```bash
SDK_DIR="$HOME/.openwrt/sdk/24.10.6"
rsync -a --delete --exclude .git "$PWD/AdGuardHome/" "$SDK_DIR/package/AdGuardHome/"
cd "$SDK_DIR"
make defconfig                                   # 让 OpenWrt 识别该包
make -j"$(nproc)" package/AdGuardHome/compile V=s
```

产物：`bin/packages/x86_64/base/AdGuardHome_0.107.79-r1_x86_64.ipk`（约 11MB，内含 30MB 静态二进制）。
实测本机耗时 **约 4 分钟**（npm ci 13s + webpack 56s + Go 编译 + 打包）。

Makefile 可调变量（见文件头部注释）：`ADGH_GO`、`ADGH_NODE_DIR`、`ADGH_NPM`、
`ADGH_NPM_REGISTRY`（默认 npmmirror）、`ADGH_GOPROXY`（默认 goproxy.cn）。
安装路径与 luci 插件一致：`/etc/config/adGuardConfig/AdGuardHome`，并软链到 `/usr/bin/AdGuardHome`。
架构映射（`ARCH`→`GOARCH`）已实测：x86_64→amd64、aarch64→arm64、arm→arm(GOARM)、
mipsel→mipsle(softfloat)、mips64el→mips64le(softfloat)、riscv64。

> 注意：packages feed 里还有一个同名的 `adguardhome` 包，与本包功能重复，**不要同时安装**。

## 常见问题

- **没有 Dev Container 图标/入口**：IDEA ≥ 2025.1 且为 Ultimate；启用 Docker 插件后重启 IDE。
- **报 Docker Desktop 相关要求**：JetBrains 官方只支持 Docker Desktop；先
  `docker context use desktop-linux`（装了 Docker Desktop 的话）排除 OrbStack 兼容性问题。
- **容器内改文件提示只读**：确认 Dockerfile 里 `openwrt` 的 UID 等于宿主 `id -u`；
  或临时把 `devcontainer.json` 的 `remoteUser` 改为 `"root"` 排查。
- **编译报错找不到依赖 / feed**：feeds 没装全：`cd ~/.openwrt/sdk/<版本> && ./scripts/feeds update -a && ./scripts/feeds install -a`。
- **feeds 克隆极慢**：见上文“安装 SDK”里的 GitHub 镜像替换方法。
- **编译慢**：确认用 `make -j"$(nproc)"`。首次构建 kernel/kmod/固件基础链在模拟器下慢属正常；
  之后约 3.5 分钟/次，且这部分是**模拟环境下遍历依赖目标的固定开销**（约 28 个目标 × 7s），
  不是真的在重编。若想进一步压缩：本包的 `DEPENDS:=+!wget&&!curl:wget` 在“wget/curl 都未选中”时
  会拉入整条 wget 依赖链（libustream-ssl/openssl/wolfssl/uclient 等）；在 menuconfig 里选中
  `curl` 可让该条件依赖落空、显著缩短闭包——代价是 ipk 的 `Depends` 字段会与 CI 产物不同，按需权衡。
- **BUILDBOT**：默认 `y` 会在每轮编译后清空各包构建目录。已在本机 SDK 中关闭
  （`Config-build.in` 的 `default n` + `.config` 里的 `# CONFIG_BUILDBOT is not set`），
  新 SDK 按“安装 SDK”第 4 步操作即可；关掉后中间产物保留，便于排查与增量编译。
- **.lmo 译文缺失**：执行 `make package/feeds/luci/luci-base/host/compile` 生成 po2lmo 后重新编译。
- **SDK 重建/版本切换**：换版本就换 `SDK_VER` 与 URL 另解压一份（SDK 目录按版本隔离在 `~/.openwrt/sdk/<版本>`）。
