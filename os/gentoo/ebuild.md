# Ebuild 编写指南

## 必须有的字段

```bash
EAPI=8                 # 必填，当前推荐 8
DESCRIPTION="..."      # 一句话描述
HOMEPAGE="..."         # 项目主页
SRC_URI="..."          # 源码地址（live ebuild 用 EGIT_REPO_URI 代替）
LICENSE="MIT"          # 必须是仓库中已存在的 license
SLOT="0"               # 必填，多数软件写 0
KEYWORDS="~amd64"      # 必填，不接受稳定化可写 ""
RDEPEND=""             # 习惯上必写，哪怕为空
```

少任何一个 `pkgcheck` 都会报错。`IUSE`、`DEPEND`、`BDEPEND` 视情况可省。

## 依赖变量分工

| 变量 | 用途 |
|---|---|
| `RDEPEND` | 运行时依赖 |
| `DEPEND` | 编译时依赖（现在通常 `DEPEND="${RDEPEND}"`） |
| `BDEPEND` | 构建工具（cmake、gettext、jq 等），只在构建机上跑 |

- `virtual/zlib:=` 中的 `:=` 表示 ABI 变动时重编依赖它的包。
- USE 开关写法：`readline? ( sys-libs/readline:= )`。

## 常用 inherit eclass

- `cmake`：提供 `cmake_src_configure/compile/install/prepare` 四件套，
  `src_configure` 里定义 `local mycmakeargs=(...)` 会被自动读取。
- `git-r3`：拉 `EGIT_REPO_URI` 源码，live ebuild 版本号约定为 `9999`。
- `plocale`：配合 `PLOCALES` 做多语言裁剪。
- `linux-info` / `toolchain-funcs` 等按需引入。

## 阶段函数（按 emerge 执行顺序）

### src_prepare
打补丁、删文件、sed 改源码：
```bash
default                       # EAPI 8 中执行默认准备流程
sed -i 's/old/new/' file || die
```

### src_configure
```bash
local mycmakeargs=(
    -DFOO="$(usex foo)"
)
cmake_src_configure
```
`usex foo`：USE 开启输出 ON，未开输出 OFF，一个顶一对 if。

### src_compile
```bash
default
use nls && cmake_src_compile lang   # 附加构建目标
```

### src_install
装到 `${D}`（不是 `/`）：
```bash
cmake_src_install
einstalldocs                        # 比手写 dodoc 省事
```

### src_test
配合 `RESTRICT="!test? ( test )"`,未开 `test` USE 时跳过测试阶段。

## 路径与变量

| 变量 | 含义 |
|---|---|
| `${P}` | name-version |
| `${PN}` | 包名 |
| `${PV}` | 版本号 |
| `${S}` | 解压后的源码目录 |
| `${D}` | 安装暂存目录（image） |
| `${WORKDIR}` | 工作目录 |

`SRC_URI` 尽量用 `${P}` 拼接，升版本只改一行。

## 注意事项

1. **`|| die` 不能省**：手写命令失败必须中止，否则 emerge 带着坏状态继续。
2. 不要 `cd` 出 `${S}`，部分 eclass 假定当前目录。
3. `IUSE` 中默认开启的开关写 `+foo`。
4. `${LINGUAS-uk}` 这种 bash 展开可用于「变量未定义时兜底」的判断。
5. 变量引用推荐 `${var}` 形式，防歧义。

## 检查与提交流程

```bash
pkgcheck scan      # 静态检查（推荐，替代老的 repoman）
pkgdev commit      # 生成 Manifest + 提交一条龙
```

## 已知坑

- **Manifest 必须生成**：overlay 仓库的 ebuild 缺 Manifest 且未推送 GitHub 时,
  `git pull` 拉取远程可能把本地新版本（如 sdcv-9999）覆盖丢失，被 `aux_get` 跳过。
  Manifest 是维护者产物而非构建产物，记录 DIST 文件校验和。
- **别抄重复的 `KEYWORDS`**：sdcv ebuild 里先写 `KEYWORDS=""` 后写 `KEYWORDS="~amd64"`,
  bash 后者覆盖前者，实际生效 `~amd64`，只留一行。
- **autounmask 污染 git**：若 `/etc/portage/package.use` 是指向配置仓库的软链,
  Portage autounmask 自动追加的条目会直接写进 git 仓库，需手动挪到合适文件再提交。

## 示例：live ebuild 骨架（sdcv 风格）

```bash
# Copyright 1999-2026 Gentoo Authors
# Distributed under the terms of the GNU General Public License v2

EAPI=8
inherit cmake git-r3 plocale

DESCRIPTION="Console version of Stardict program"
HOMEPAGE="https://dushistov.github.io/sdcv/"
EGIT_REPO_URI="https://github.com/Dushistov/sdcv.git"

LICENSE="GPL-2"
SLOT="0"
KEYWORDS="~amd64"
IUSE="darkterm nls readline test"
RESTRICT="!test? ( test )"

RDEPEND="
    >=dev-libs/glib-2.36
    virtual/zlib:=
    readline? ( sys-libs/readline:= )
"
DEPEND="${RDEPEND}"
BDEPEND="
    nls? ( >=sys-devel/gettext-0.14.1 )
    test? ( app-misc/jq )
"

src_configure() {
    local mycmakeargs=(
        -DENABLE_NLS="$(usex nls)"
        -DWITH_READLINE="$(usex readline)"
        -DBUILD_TESTS="$(usex test ON OFF)"
    )
    cmake_src_configure
}

src_compile() {
    cmake_src_compile
    use nls && cmake_src_compile lang
}

src_install() {
    cmake_src_install
    einstalldocs
}
```

## Rust（cargo）包

### 顺序铁律

bash 赋值是执行到那一行时立刻展开的，`inherit cargo` 在被 inherit 的那一刻
读 `CRATES` 并生成 `CARGO_CRATE_URIS`，所以：

- **`CRATES` 必须写在 `inherit cargo` 之前**——否则 eclass 读到空列表，
  展开出空的 URL。
- **`SRC_URI="${CARGO_CRATE_URIS}"` 必须写在 `inherit cargo` 之后**——
  否则执行到时变量还没生成，SRC_URI 被赋成空字符串。

一个前一个后，谁都不能换边。两头顺序错了症状一样：crate 全没下载，
`cargo_src_unpack` 解出空 vendor 目录，`cargo_src_compile` 带 `--offline`
 报 `no matching package named xxx found`。

### 模板

```bash
EAPI=8
DESCRIPTION="..."
HOMEPAGE="..."
EGIT_REPO_URI="https://github.com/xxx/yyy.git"    # 或 SRC_URI 指 tarball

CRATES="                    # pycargoebuild 生成，必须在 inherit 前
	foo@1.2.3
	bar@0.4.5
"

LICENSE="MIT"
LICENSE+=" Apache-2.0 BSD-2 ISC ..."              # 依赖 crate 的协议全列
                            # 由 pycargoebuild 生成，漏了 pkgcheck 报警告

inherit git-r3 cargo         # 这一刻：CRATES → CARGO_CRATE_URIS

SRC_URI="${CARGO_CRATE_URIS}"   # 必须在 inherit 后

src_unpack() {
	git-r3_src_unpack
	cargo_src_unpack         # 把 crate 全解到 vendor 目录 + 生成离线配置
}

src_compile() {
	cargo_src_compile --bin miyu   # 自带 --offline，只认 vendor 目录
}

src_install() {
	dobin "$(cargo_target_dir)/miyu"   # cargo.eclass 提供的函数，非变量
}
```

要点：

- `cargo.eclass` 认 `name@version` 也认 `name-version`（源码里 `*@*` 分支）。
- 生成 CRATES：`emerge app-portage/pycargoebuild`，需在 `portage` 组里
  （写 `/var/cache/distfiles`），然后在源码目录跑 `pycargoebuild -C ./`。
  它顺带把全部 crate 下载进 distfiles，emerge 时不用再下。
- 上游 `Cargo.lock` 变了（升依赖）就要重新生成 CRATES + 重跑 manifest。
- Rust 不读 make.conf 的 `*FLAGS`，加 `QA_FLAGS_IGNORED="usr/bin/xxx"`
  消音；上游 `strip = true` 会在 src_prepare 里
  `export CARGO_PROFILE_RELEASE_STRIP=false` 让 Portage 自己剥。
- 构建期沙箱无网络：走 cargo.eclass 全程离线，无需关 sandbox；
  不想列 CRATES 的话退路是裸 `cargo build` + package.env 关
  `-network-sandbox -sandbox`，但那套是个人 overlay 的省事做法，
  上送 tree 必须回 cargo.eclass。
