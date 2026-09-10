> [ReSukiSU-SUSFS 落地内核树指南](#ReSukiSU-SUSFS 落地内核树指南)

# 生成 .config 的经验指南（Rodin / Xiaomi_Rodin_Kernel_Enhance）

本指南总结在用 `defconfig + gki_defconfig` 重新生成 `.config` 并基于 clang22 全量编内核时踩过的坑与解法，供以后复用，避免重复踩坑。

适用内核树：`~/rodin-build/kernel-rodin-merge`（Xiaomi Rodin 6.6.x 分支 `bsp-rodin-kernel-oss-bp`）
编译器：Android 预编译 clang22：`$HOME/rodin-build/prebuilts/clang/host/linux-x86/clang-r596125/bin`

---

## 1. 推荐的 config 生成与锁choice流程

Kconfig 的 `merge_config.sh` 之后会做“求解/重整”，**会清掉依赖不满足的符号、并把“无显式选择”的 `choice` 落回默认成员**。因此不要“merge 完就跑”，要**先求解、再锁定关键 choice、再 syncconfig**：

```bash
cd ~/rodin-build/kernel-rodin-merge

# 基准 defconfig + 叠加 gki_defconfig 合并
scripts/kconfig/merge_config.sh -O .

# 关键：merge 的求解会把 LTO 这个 choice 拽回默认 LTO_NONE
# 用 scripts/config 显式锁定，避免被覆盖
./scripts/config --enable LTO_CLANG_THIN   # 同时自动让 LTO_NONE/LTO_CLANG_FULL 变 n
./scripts/config --enable AUTOFDO_CLANG    # AutoFDO
./scripts/config --enable CFI_CLANG        # 若需要 CFI
./scripts/config --enable CFI_PERMISSIVE
./scripts/config --set-str CONFIG_LOCALVERSION "-by-Omachirimanbu-4k"   # 若需 localversion
./scripts/config --set-val CONFIG_FRAME_WARN 4096                        # 见 §3

# 用 syncconfig 物化 include/config 派生头文件（不要再跑 olddefconfig！会冲掉锁定项）
make ARCH=arm64 LLVM=1 syncconfig
```

> `syncconfig` vs `olddevconfig` 区别：
> - `olddefconfig`=“重新求解 `.config`（填默认值）”，会覆盖未被显式指定的符号、把 choice 落回默认 → **会把 LTO 冲掉，别在 merge 后用它**。
> - `syncconfig`=“把当前 `.config` 物化成 `include/config/auto.conf`、`autoconf.h` 等”，不动你的选择。
> 日常 `make` 内部自动跑的是 `syncconfig`。

---

## 2. LTO 为何容易变成 LTO_NONE

内核 `LTO_CLANG_*` 是一个 `choice`（单选互斥），其 `default` 成员是 `LTO_NONE`。

- `defconfig`（基准仓）通常**不设置任何 LTO 选项**；
- `merge_config.sh` 求解时若“没有任何成员被显式且依赖满足地选中”，choice 就会收敛到默认成员 **`LTO_NONE=y`**；
- 于是即便 `gki_defconfig` 里写了 `LTO_CLANG_THIN=y`，也可能被冲掉。

解法就是上面用 `scripts/config --enable LTO_CLANG_THIN` 显式锁定，再用 `syncconfig` 物化，顺序别反。

---

## 3. clang22 下 WERROR 与栈帧告警

新版 clang 会把 `-Wframe-larger-than` 在 `CONFIG_WERROR=y` 时把栈帧超限当成 error（如 `lib/crypto/curve25519-hacl64.c`、`drivers/block/loop.c` 栈帧 2592/2740 > 默认 2048）。

解法（保留 WERROR 完整检查、仅放宽栈帧上限）：
```
CONFIG_FRAME_WARN=4096
```
配合 `./scripts/config --set-val CONFIG_FRAME_WARN 4096`。

---

## 4. AutoFDO（编译层面验证是否生效）

- 打开：`CONFIG_AUTOFDO_CLANG=y` + `CONFIG_LTO_CLANG_THIN=y`。
- 内核会 `include scripts/Makefile.autofdo`，并默认用树内 profile `android/gki/aarch64/afdo/kernel.afdo`。
- 验证是否生效：用 `V=1` 重新编译单个对象看命令行是否含：
  ```
  -fprofile-sample-use=./android/gki/aarch64/afdo/kernel.afdo
  -fdebug-info-for-profiling
  -mllvm -enable-fs-discriminator=true
  ```
- 注意：默认 profile 是 **Google android15-6.6 GKI** 的；对 Rodin 特有代码收益有限但不会报错。要用专属 profile 请自采样 `.afdo` 后 `CLANG_AUTOFDO_PROFILE=xxx` 重编。
- `strings vmlinux | grep profile` 查不到是正常的（`-fprofile-sample-use` 不把字符串嵌进最终镜像），不能以此判断失效。

---

## 5. 编译相关的小patch（这些改动已提交到 bsp-rodin-v-oss-bp）

| 改动 | 文件 | 原因 |
|---|---|---|
| `#include <linux/sched/task_stack.h>` | `drivers/kernelsu/feature/adb_root.c` | LTO(clang22)下 `undefined symbol: task_stack_page`（static inline 未内联成外部符号）；引入该头可在 TU 内内联。**不要直接去 static/加 extern**，会引入 duplicate symbol。 |
| `fanotify_fdinfo` 增加 3 参回调变体（`CONFIG_KSU_SUSFS_SUS_MOUNT`） | `fs/notify/fdinfo.c` | SUSFS 打开后 fanotify 回调用旧 2 参，与 3 参回调不匹配编译失败（参考 inotify 写法）。 |

## 6. 编译命令模板

```bash
export PATH="$HOME/rodin-build/prebuilts/clang/host/linux-x86/clang-r596125/bin:$PATH"
cd ~/rodin-build/kernel-rodin-merge
make ARCH=arm64 LLVM=1 -j8 Image Image.lz4
```

- 产物：`arch/arm64/boot/Image`（未压缩）、`arch/arm64/boot/Image.lz4`（LZ4 legacy 压缩）。
- 验证 lz4 legacy：`xxd arch/arm64/boot/Image.lz4 | head -1` 魔数应为 `0221 4c18`（`0x184C2102`），即 LZ4 Legacy，**不是** `0422 4d18`（标准帧格式）。
- 版本号在 `include/config/kernel.release`。

---

## 常见坑速查

| 现象 | 原因 | 解决 |
|---|---|---|
| merge 后大量 defconfig 符号消失 | Kconfig 求解按依赖裁剪（ARCH_QCOM/TEGRA/SUNXI 等置 n ⇒ 其驱动符号一起没了） | 正常现象，不追回；若需要某驱动就把其 `CONFIG_*` 与父 `ARCH_*` 一起 `scripts/config --enable` |
| LTO 变成 NONE | choice 默认成员 | `scripts/config --enable LTO_CLANG_THIN` 后 `syncconfig` |
| clang 栈帧超限编不过 | clang22 + WERROR | `CONFIG_FRAME_WARN=4096` |
| `task_stack_page` 未定义 | adb_root.c 未含头文件 | 顶部加 include |
| SUSFS 下 fdinfo 编译失败 | fanotify 回调签名不匹配 | fdinfo.c 补 3 参分支 |

---

# ReSukiSU-SUSFS 落地内核树指南：头文件重建与版本号计算

> 适用场景：把最新 `resukisu-susfs`（ReSukiSU Fork）的 kernel 模块源码「完整落地」到内核树
> `kernel-rodin-merge` 的 `drivers/kernelsu`，并在落地后用 `fix_kbuild.py` 固定正确的版本号。
>
> 本文档基于一次完整落地实践总结，包含两个易踩坑的环节：
> 1. **`uapi/` 头文件重建**（落地后编译必现，缺了会找不到头文件）
> 2. **版本号计算**（依赖 `fix_kbuild.py` 固化版本信息）

---

## 0. 路径约定

以下路径可按照实际环境替换。本文使用 WSL 中的路径：

| 含义 | 路径 |
|------|------|
| 源仓库（ReSukiSU-susfs） | `/home/qwe12/rodin-build/resukisu-susfs` |
| 内核树（目标落地位置） | `/home/qwe12/rodin-build/kernel-rodin-merge` |
| 源模块源码目录 | `<源根>/kernel` |
| 落地目标目录 | `<内核根>/drivers/kernelsu` |
| 版本修正脚本（Windows 侧） | `fix_kbuild.py` |

> Windows 与 WSL 路径映射：`\\wsl.localhost\Ubuntu-24.04` 对应 `/`，例如
> `\\wsl.localhost\Ubuntu-24.04\home\qwe12\rodin-build\...` == `/home/qwe12/rodin-build/...`。

---

## 1. 完整落地源码到内核树

### 1.1 清空并重新拷贝

落地前先清理旧目录，再用 `rsync --delete` 让目标严格等于源目录：

```bash
cd <内核根>/drivers
rm -rf kernelsu                         # 清理旧版本，避免残留旧文件
rsync -rv <源仓库>/kernel/ ./kernelsu/  # 完整同步（保留符号链接与权限）
```

> 要点：`rsync` 配合 `--delete`（或 `-rv` 的删除语义）时，目标会删除源中不存在的文件，
> 所以源里被删掉的文件会在目标里被同步删掉——这正是第 2 节 `uapi/` 缺失问题的来源。

### 1.2 同步其它内核侧 SUSFS 文件

落地不只是拷贝 `drivers/kernelsu`，SUSFS 内核文件也要与源保持一致（落地完成后验证）：

```bash
diff <源根>/susfs4ksu/kernel_patches/include/linux/susfs.h  <内核根>/include/linux/susfs.h
diff <源根>/susfs4ksu/kernel_patches/fs/susfs.c              <内核根>/fs/susfs.c
# susfs_def.h 同理：来自 susfs4ksu/kernel_patches/include/linux/susfs_def.h
```

### 1.3 内核侧补丁（50/55/56）状态

- 内核侧补丁（如 `50_add_susfs_in_ath-6.1.patch`、`55_selinux_hook_hide...`、
  `56_selinuxfs_hide...`）来自 `susfs4ksu/kernel_patches/`。
- 若这些补丁**已打在树里**，再落地时不需要重复 `git apply`（会因已应用而失败）。
- 可用 `git apply --check` 判断；失败通常意味着已应用。

> 注意：`susfs4ksu/kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch` 是给**标准**
> KernelSU 用的。resukisu-susfs 的 `kernel/` 本身就内置 SUSFS（Kconfig 有 20 个
> `KSU_SUSFS` 项），因此**不要**对这个落地源再应用该补丁。

---

## 2. 头文件重建（关键坑）

### 2.1 问题现象

落地并 `rsync --delete` 后，会有如下报错表象：

```bash
git status --short               # 出现 D drivers/kernelsu/uapi/...
grep -rn 'uapi/' drivers/kernelsu  # 仍有多个源文件引用 "uapi/xxx.h"
```

即：落地后源文件仍 `#include "uapi/app_profile.h"` 等，但 `drivers/kernelsu/uapi/`
目录没了，编译会报找不到头文件。

### 2.2 根因

源仓库里，`kernel/`（模块源码）自身**并没有 `uapi/` 目录**，头文件在**仓库根目录的
`uapi/`**：

```text
resuki-susfs/
├── kernel/                  # 落地为 drivers/kernelsu
│   └── include/
│       └── uapi -> ../../uapi   # 软链接：指向仓库根目录 uapi/
└── uapi/                    # 真正头文件在这里
    ├── app_profile.h
    ├── feature.h
    ├── ksu.h
    ├── selinux.h
    ├── sulog.h
    └── supercall.h
```

源码 `kernel/include/uapi -> ../../uapi` 是相对软链接，在源仓库里指向仓库根的
`uapi/`；但 `kernel/` 被复制进内核树成 `drivers/kernelsu` 后，它的解析路径变成
`drivers/uapi`（**不存在**），即相对链接失效。

而内核侧 Kbuild 的包含路径是：

```makefile
ccflags-y += -I$(srctree)/$(src) -I$(srctree)/$(src)/include
```

也就是 `-Idrivers/kernelsu`，所以 `#include "uapi/xxx.h"` 实际解析到
`drivers/kernelsu/uapi/xxx.h`——需要一个**真实的 `drivers/kernelsu/uapi/` 目录**。

### 2.3 修复：重建真实 uapi/ 目录

把仓库根目录的 `uapi/*.h` 拷进内核树，建一个真实目录（不用软链接）：

```bash
SRC=<源根>/uapi
DST=<内核根>/drivers/kernelsu/uapi
mkdir -p "$DST"
cp "$SRC"/*.h "$DST/"
```

落地后应包含 6 个 `.h`：

```bash
ls <内核根>/drivers/kernelsu/uapi/
# app_profile.h  feature.h  ksu.h  selinux.h  sulog.h  supercall.h
```

### 2.4 include/uapi 软链接

`drivers/kernelsu/include/uapi -> ../../uapi` 在内核树里会解析到 `drivers/uapi`
（不存在），是**无效软链接**，对编译无作用：

- 保留即可（与 git 已提交状态保持一致）；
- 真实头文件由 `drivers/kernelsu/uapi/` 真实目录提供。

---

## 3. 版本号计算与修正

### 3.1 版本号来源（从源仓库 git 读取）

版本号公式（内核 Kbuild 原始定义）：

```text
KSU_VERSION = 30000 + KSU_LOCAL_VERSION + 700
```

需要 4 个字段，全部来自**源仓库**（非内核树）：

```bash
cd <源根>   # 例如 /home/qwe12/rodin-build/resukisu-susfs

KSU_LOCAL_VERSION=$(git rev-list --count HEAD)                        # 例 4427
KSU_TAG_NAME=$(git describe --abbrev=0 --tags 2>/dev/null || echo v4.1.0)  # 例 v4.2.0-rc1
KSU_COMMIT_SHA=$(git rev-parse --short=8 HEAD)                        # 例 23a40c0f
KSU_BRANCH_NAME=$(git branch --show-current)                          # 例 main-susfs

KSU_VERSION=$((30000 + KSU_LOCAL_VERSION + 700))                      # 例 35127
```

### 3.2 为什么不能用内核树自己的 git（fix_kbuild.py 存在的意义）

Kbuild 默认行为是从 `$(KSU_SRC)` 查 git 拿这些值。但落地后：

- `drivers/kernelsu/` 本身已是内核树 git 仓库的一部分；
- 若让内核树去查询，读到的是**内核树的 commit 数/sha/脏状态**，而非 ReKSU 的。

因此 `fix_kbuild.py` 把 Kbuild 的 git 查询分支替换为：`LOCAL_GIT_EXISTS=1` 时查
git（外部构建），否则**直接写入固化的版本号**。

### 3.3 fix_kbuild.py 用法

脚本是**幂等**的文本替换器，落地完成后运行：

```bash
python3 fix_kbuild.py
# 输出 "ALREADY PATCHED"（已打过）或 "PATCHED OK"（刚打好）
# 若 old 块未找到会 abort（通常表示 new 已在）
```

它会：
1. 用 `old` 块匹配内核 Kbuild 中原始的 git 查询段；
2. 替换为 `new` 块（固化版本号 + `ifeq/else/endif` 分支）；
3. 我已修复判断顺序，优先检测 `new` → 输出 `ALREADY PATCHED`，避免误报 `ABORT`。

替换后 Kbuild 应含：

```makefile
ifeq ($(LOCAL_GIT_EXISTS),1)
$(shell cd $(KSU_SRC); ...)
KSU_LOCAL_VERSION := $(shell ...)
...
else
# Vendored as an in-tree copy of ReSukiSU main-susfs @ 23a40c0f. ...
KSU_LOCAL_VERSION := 4427
KSU_TAG_NAME    := v4.2.0-rc1
KSU_COMMIT_SHA  := 23a40c0f
KSU_BRANCH_NAME := main-susfs
endif
KSU_VERSION := $(shell expr 30000 + $(KSU_LOCAL_VERSION) + 700)
```

### 3.4 验证版本

```bash
grep -n 'KSU_VERSION\|KSU_LOCAL_VERSION\|KSU_TAG_NAME\|KSU_COMMIT_SHA\|KSU_BRANCH_NAME' \
  <内核根>/drivers/kernelsu/Kbuild
```

期望：`KSU_LOCAL_VERSION := 4427`、`KSU_COMMIT_SHA := 23a40c0f`、
`KSU_VERSION := ...30000 + 4427 + 700`（`35127`）。

---

## 4. 落地后自检清单

| 检查项 | 命令 |
|--------|------|
| 无未跟踪产物 | `git status --short \| grep '^??'` 为空 |
| 无意外删除 | `git status --short \| grep '^ D'` 为空 |
| hook 目录完整 | `ls drivers/kernelsu/hook/`（arm32/arm64/x86_64 + 钩子）|
| ksu.h 含版本宏 | `grep KERNEL_SU_VERSION drivers/kernelsu/include/ksu.h` |
| uapi 头齐备 | `ls drivers/kernelsu/uapi/`（6 个 .h） |
| 版本固化 | 见 3.4 |

通过后 `git add -A && git commit`。

---

## 5. 本次最终值（供参考）

| 项 | 值 |
|----|----|
| 源分支 | `main-susfs` |
| 源 commit | `23a40c0f` |
| tag | `v4.2.0-rc1` |
| KSU_LOCAL_VERSION | `4427` |
| KSU_VERSION（固定值） | `35127` |
| 落地提交 | `a5724993c054` |