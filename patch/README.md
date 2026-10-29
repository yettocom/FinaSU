# FinaSU 三星补丁（Samsung S23U / KDP + RKP + DEFEX）

FinaSU 上游本身不带三星支持：在真机上，`ksu_patch_text()` 的 fixmap 写入会被 EL2 层
（RKP / proca）拒绝，cred 计数又被 KDP 写保护，DEFEX 还会把 KSU 域进程当提权目标。
本目录是让 FinaSU 在这些机型上可用的补丁集。

## 当前版本

| 补丁 | 适用基线 | 大小 | 文件数 | 内容 |
|---|---|---|---|---|
| `FinaSU-v1.2.1-samsung-kdp-rkp-defex.patch` | FinaSU **v1.2.1**（tag `v1.2.1` = `be19eb90`，33074） | 100,641 B | 20 | 内核侧：RKP 分支补丁器、KDP cred 提交、DEFEX 同步、LSM/syscall 钩子接入 |
| `FinaSU-v1.2.1-samsung-lateload.patch` | 同上 | 17,197 B | 3 | userspace/ksud：late-load 时先把 daemon 落地，再加载模块；加载完成后补跑 `boot_hide::boot_apply()` |

本目录只保留当前版本的两份补丁（v1.2.0 的补丁与 v1.2.1 逐字节相同的部分见文末第二十三轮），
旧版本请用 `git log -- patch/` 取回。
`FinaSU-v1.0.9-*`（102,344 B / 16,613 B）、`FinaSU-v1.0.7-*`（90,944 / 16,613）与
`FinaSU-v1.0.5-*` 保留给对应旧基线（`f5ad2e83` = 33023；`ef67206b` = 32997；
`3f06a69d` = 32990）。各版本补丁是同一套改动重放到对应基线的 SUSFS 集成形态
（`#ifdef CONFIG_KSU_SUSFS` 分段包裹），语义等价。

## 应用

```sh
git -C <finasu-src> checkout <tag>          # v1.2.1 = be19eb90
git -c core.autocrlf=false apply FinaSU-v1.2.1-samsung-kdp-rkp-defex.patch
git -c core.autocrlf=false apply FinaSU-v1.2.1-samsung-lateload.patch
```

两份补丁互不重叠，可以只应用内核那份（纯内核改动），也可以只应用 ksud 那份。

## 验证

```sh
bash D:/root/finasu-work/_verify121.sh       # 生成 1.2.1 两份补丁并做正向/反向逐字节校验
```

1.2.1 的三 KMI 构建（`_build121.sh`，`warnings=0`、`-- KernelSU version: 33074`，
版本串 `v1.2.1-be19eb90@Vergissmeinnicht`）：

| KMI | ko | sha256 |
|---|---|---|
| android13-5.15 | 626,296 B | `58ed47bae73b3e74…` |
| android14-6.1 | 648,568 B | `88549a285a9fcf1f…` |
| android15-6.6 | 512,344 B | `984dd78ec31d4aac…` |

## 改了什么（内核侧）

- `kernel/compat/samsung_rkp.c`（新增）：RKP 机型上不写系统表项，改为在函数入口写一条 `B`
  跳到自己 16 B 的 stub；stub 保留 kCFI type hash、landing pad（BTI C / PAC）、被搬移的指令，
  再跳回原函数。`ksu_samsung_rkp_branch_patch_callthru()` 在分支生效前先把可调用入口发布给
  调用方（callthru），调用方在 RKP 上走 callthru 而不是原来那份被覆盖的指针。
- `kernel/compat/samsung_kdp.c`（新增）：通过 `prepare_ro_creds()` / `kdp_assign_pgd()` 提权，
  而不是直接改 `cred`（KDP 写保护的页上直接写会触发外部中止）；cred 计数改用
  `kdp_usecount_*` 系列。符号缺一个就整体降级为标准路径。
- `kernel/compat/samsung_defex.c`（新增）：两个 kprobe 让 `task_defex_enforce`/`task_defex_user_exec`
  对 KSU 域与 root 任务放行，并在提权后同步 DEFEX 自己的 cred 三元组（uid/fsuid/egid）。
- `kernel/hook/arm64/patch_memory.c`：`ksu_patch_text()` 在 8 字节（函数指针槽）写入且 RKP 活跃时
  转发给分支补丁器。
- `kernel/hook/arm64/syscall_hook.c`：dispatcher 表项安装同样支持 callthru 路线；装完回读校验，
  失败不登记。
- `kernel/feature/selinux_hide.c`：`write_op`/`setprocattr`/`status open` 三个钩子全部支持 callthru；
  还原点只在还原成功后清除；`enable()` 入口先自愈上一次失败的还原（否则会把自有函数当原函数）。
- `kernel/policy/app_profile.c`：`escape_with_root_profile()` 改走 `ksu_samsung_kdp_commit_creds()`，
  失败即放弃（不再带着半套凭据继续 exec），成功后补 DEFEX 同步。
- `kernel/selinux/selinux.c`：`is_sid_match()` 的字符串回退会分配内存，在 DEFEX kprobe
  pre-handler（原子上下文）里不可用——加 `in_atomic() || irqs_disabled()` 提前返回。
- `kernel/core/init.c`：初始化顺序改为 `symbol_resolver → rkp → kdp → creds → defex → 各 feature`，
  退出按反序拆。
- `kernel/Kbuild`/`Kconfig`：arm64 上 `CONFIG_KSU_SAMSUNG_KDP/DEFEX` 默认 `y`（out-of-tree 构建
  不读 Kconfig，不显式传就会静默编成空桩）。

## 改了什么（ksud 侧）

- `late_load.rs`：加载模块前先 `stage_daemon_from("/data/local/tmp/.ksud-stage")`——进 KernelSU
  域之后 `/data/adb/ksud` 就写不动了；`bundled=1` 让管理器显示为自带 LKM。
- `utils.rs`：`stage_daemon_from()` 只接受 root 拥有、组/他人不可写的普通文件；用同目录临时名
  原子替换（不原地截断，避免覆盖正在执行的 daemon）；`EXDEV` 时退化为拷贝；校验失败回退
  `/proc/self/exe`，不再让设备失去 daemon。

## 已知边界

- 只覆盖 arm64；非 arm64 上 `samsung_rkp.c` 编译成桩（`-EOPNOTSUPP`），行为与上游一致。
- RKP 站点表 16 槽：dispatcher + read + fstat + selinux_hide 4 处 + 5 个 LSM 钩子 = 12，余量 4。
- `escape_to_root_for_init()` 仍用裸 `commit_creds()`：init 不是 KDP 目标，改了反而要处理
  vendor 语义未验证的风险；只在真机上复核过这一条路径。
- KDP 提交失败会让提权直接失败（fail-closed）；上游是 fail-open，这里是有意选择。
- Exynos 机型需要另配 `SUSFS_SAMSUNG_NO_PATCH_TEXT` 之类的开关，本补丁没有覆盖。

## 更长的背景

- 逐轮审计记录：`D:\root\_map\learn\AUDIT-SUMMARY.md`、`DEBUG2-SUMMARY.md`、`FINASU-107-PATCH-STYLE-AUDIT.md`
- 三引擎/挂载/发版流程：`D:\root\_map\01-finasu.md`、`D:\root\_map\06-patch-release.md`
- 历史 HTML/文本对话提炼：`D:\root\_map\learn\H1-history.md`

## 第十五轮：采纳 diabl0w 的 syscall 帧修复（2026-10-23）

对照上游 fork（`diabl0w/KernelSU` main `76a088c5`）发现一处真实缺陷并采纳：

- `kernel/hook/syscall_hook_manager.c` 的 `ksu_sys_enter_handler(void *data, struct pt_regs *regs, long id)`
  原来用 `task_pt_regs(current)` 写 `syscallno` 与 `regs[8]`。但 tracepoint 的契约是
  「第一参就是本次 syscall 的入口帧」：`include/trace/events/syscalls.h:18-22` 的
  `TRACE_EVENT_FN(sys_enter, TP_PROTO(struct pt_regs *regs, long id), ...)`，
  由 `kernel/entry/common.c:77` 的 `trace_sys_enter(regs, syscall)` 传入；
  而 `task_pt_regs(p)`（`arch/arm64/include/asm/processor.h:273`）是「任务内核栈顶那一帧」，
  只是**常常**与入口帧相同，并不是同一回事——帧不等价时会写错内存（diabl0w 在 A065 上实测崩溃，
  提交 message：「Fix A065 crash: use regs param instead of task_pt_regs(current)」）。
- 现在直接使用入参 `regs`（aarch64 `PT_REGS_ORIG_SYSCALL(regs)` + `regs->syscallno`，x86_64
  `regs->ax/orig_ax`，riscv `ksu_riscv_redirect_syscall(regs, ...)`），并删掉只为这个用法存在的
  `#include <linux/sched/task_stack.h>`。失败模式也从「写坏任意内存」变成「不重定向」。
- 未采纳的同类改动：`76a088c5` 把 landing-pad 掩码改成显式 switch 枚举（纯可读性，语义等价）；
  `tiann/KernelSU` 与 `KernelSU-Next` 的同一处仍是 `task_pt_regs`，这个修复目前只有 fork 里有。

后端到端复核：`git diff` 后 1 文件 +13/−8、新增行 `goto` 计数 0、最长行 90 列、非 ASCII 0；
三 KMI 重编 0 warning（哈希见上表），补丁重生成 90,944 B / 18 文件 / +1,768 −85，
`_verify107.sh` 正向+反向逐字节通过。

配套发布件：`D:\root\finasu-release\1.0.7\`（三 ko + ksud 7,398,792 B + 官方 APK + DFRoot 专版），
详见该目录 `README.md` 与 `SHA256SUMS`。

## 第十六轮：基线同步到 v1.0.9（2026-10-24，按用户要求未编译）

上游 v1.0.9 = tag `f5ad2e8325d2ee5d123519a6c27f1cfe5c7da673`（commit `3b79d94e`，版本
33023）。相对 v1.0.7，我们补丁涉及的 21 个文件里只有 7 个变了：`kernel/Kbuild`、
`kernel/Kconfig`（新增整段 SUSFS menu，83 行）、`kernel/core/init.c`、`kernel/feature/
selinux_hide.c`（56 行）、`kernel/feature/sucompat.c`、`kernel/hook/syscall_hook_manager.c`、
`kernel/selinux/selinux.c`；`patch_memory.c` / `syscall_hook.c` / `tp_marker.c` /
`app_profile.c` / `ksud_integration.c` 与三个 userspace 文件逐字节未变。

- 新树 `D:\root\finasu-work\_src109`：基线 `8aabe08`（v1.0.9），补丁提交 `d99ff87`。
- 1.0.7 补丁直接 `git apply --reject` 时 11 个文件自动落地、4 个冲突（Kbuild/Kconfig/
  core/init.c/selinux_hide.c）。**冲突不用手工解 marker**，改为「以 v1.0.9 为底座、按锚点
  重放我们的语义」：脚本 `D:\root\finasu-work\_mrg109_apply.py`（每处 `sub1()` 断言命中次数，
  定义/声明用 `func()` 区分），保留上游新增的 `#ifdef CONFIG_KSU_SUSFS` 分段包裹、
  `KSU_VERSION_FULL` 打印与 `susfs_init()` 调用。
- 结果：kernel 18 文件 `+1,744/−89`（补丁 88,845 B）、userspace 3 文件 `+232/−39`
  （补丁 16,613 B），四个重放文件 CR 计数全 0。
- 校验（`_verify109.sh`）：正向 apply 两补丁后与 `_src109` **逐字节相同**；从已打补丁树
  反向 `-R` 后与 `_pristine109` **逐字节相同**；`-R --check` 两项均 ok。
  注意「反向」只在已打补丁的树上成立——补丁含新增文件，`_pristine109` 上直接 `-R` 必然失败。
- 按用户要求**本轮不编译**；构建脚本仍是 `_build107.sh`（v1.0.7 树），1.0.9 的构建待下一步。

## 第十七轮：对照 v1.0.9 源码 debug（SUSFS 分段交互 + 重放丢失项）

- 触发：按「对照源码 debug、遵循补丁原则」复核刚重放到 1.0.9 的补丁。首轮编译（修复前）
  三 KMI 已 0 warning，但暴露两类问题。
- **修复 1（构建配置交互，必须修）**：v1.0.9 把 `kernel/feature/selinux_hide.c` 的
  `selinux_write_op`/`context_write`/`access_write`/`orig_*`/`callthru_*` 放进了
  `#ifndef CONFIG_KSU_SUSFS`，而重放时新加的四个 helper 落在 guard 外并引用这些标识符 →
  `CONFIG_KSU_SUSFS=y` 构建会编译失败；同时 `#include "ksu_samsung_rkp.h"` 被上游放进 guard
  内，而 guard 外的 `hook_selinux_status_open()` 需要 `ksu_samsung_rkp_active()/get_stub()`。
  改法：include 移出 guard；`install_write_op`/`restore_write_op` 与其调用点整体放进
  `#ifndef CONFIG_KSU_SUSFS`；`restore_all()` 的 status-open 部分两配置通用。
- **修复 2（重放丢 delta）**：与 v1.0.7 已审计版逐文件 diff，补回三处——
  `my_sel_open_handle_status` 的 `__nocfi`；`ksu_selinux_hide_exit()` 的
  `else { ksu_selinux_hide_unhook(); }`（init 装过 status 钩子但从未 enable 时，卸载会留下
  指向模块已释放代码的槽）及其后的 `restore_all()` 重试 + `pr_err`；两个 helper 的理由注释。
- 复核通过（不改）：`kernel/Kbuild` 的 compat 对象与 arm64 `?= y` 默认、`kernel/Kconfig` 两段
  config 在 SUSFS menu 之前、`kernel/core/init.c` 的 rkp→kdp→prepare_creds→susfs→defex→
  syscall_hook 顺序、`sucompat.c` 的 escape 失败分支（上游已含）、`syscall_hook_manager.c`
  用 tracepoint 入参 `regs`、`selinux.c` 的 `in_atomic()` 守卫；新增行 goto 计数 0。
- 提交 `4967781 kernel: keep the samsung selinux hooks buildable under SUSFS`；补丁
  **91,286 B**（18 文件）/ 16,613 B（3 文件）；`_verify109.sh` 正反向逐字节校验全绿。
- 修复后三 KMI 编译（`_build109.sh`，`KSU_GIT_VERSION=3023`，DDK 20260828）warnings=0、
  `-- KernelSU version: 33023`：5.15 597,416 B `e6a34133…`、6.1 632,224 B `ee7b0ee9…`、
  6.6 496,344 B `87e4771a…`。

## 第十八轮：对照源码 debug（v1.0.9，提交 `2c2062c`）

**最重的一条：KDP/DEFEX 从来没真正编进去过。** `kernel/Kbuild` 只设了 make 变量
（`CONFIG_KSU_SAMSUNG_KDP ?= y`），而 C 侧判的是预处理器宏；外部模块构建不读 Kconfig，
于是三个 1.0.x 发布件的 `kernelsu.ko` 里两个兼容层全是桩——对 v1.0.5 / v1.0.7 / v1.0.9 的 ko
`grep -ac 'samsung_kdp:'` / `'samsung_defex:'` 都是 0，同期 `samsung_rkp:` 是 20。

- 修法：arm64 分支里显式 `ccflags-y += -DCONFIG_KSU_SAMSUNG_KDP=1`、
  `-DCONFIG_KSU_SAMSUNG_DEFEX=1`（与同文件 `CONFIG_KSU_DISABLE_MANAGER` 的写法一致）。
- **不要再嵌套 `ifeq ($(CONFIG_KPROBES),y)`**：外部模块构建看不到内核配置符号（DDK 树里
  没有 `include/config/auto.conf`），这一层会静默关掉 DEFEX——第一次修完 `kdp=2/defex=0`
  就是它。`linux/kprobes.h` 在无 kprobe 内核上把注册函数折成 `-ENOSYS` 桩、
  `samsung_defex_init()` 又运行时解析符号，所以「编进去但惰性」比「静默不编」可控。
- 覆盖判据（三 KMI）：`grep -ac 'samsung_defex:'` = 10、`samsung_kdp:` = 6、`samsung_rkp:` = 20。

**P1：兼容层没有退出路径。** `ksu_samsung_defex_exit()` / `ksu_samsung_rkp_exit()` 全树零调用点：
卸载后 DEFEX 的两个 kprobe 仍注册、RKP 的分支仍指向即将释放的模块 text。`kernel/core/init.c:309-310`
在 `ksu_module_blacklist_exit()` 之后、`ksu_put_cred(ksu_cred)` 之前补上（顺序即拆除顺序：先拆
所有消费者，再拆由它们持有的槽与分支）。

**P2 四处**：`hook_selinux_status_open()` 回滚失败不再清 slot/orig（留给 `restore_all()` 重试，
否则槽与分支无人跟踪）；`ksu_samsung_rkp_branch_patch_callthru()` 失败不再还桩槽（`*callthru`
在写分支之前已发布，读者可能持有 `stub+4`），并接住回滚返回值——回滚失败意味着入口 PAC 被换成
`BTI C`，尾声会 raise FPAC，只能 `pr_err` 留证据；`escape_to_root_for_adb_root()` 补
`ksu_samsung_defex_sync_current()`（与另外两条提权路径一致）；KDP worker 的 `-EBUSY`/`-EIO`
分支清 `rw_cred`（静态对象不留悬空指针）。另把 `samsung_rkp.c:319` 方向写反的注释改对。

**验证**：三 KMI `warnings=0`、`-- KernelSU version: 33023`——5.15 621,456 B `cc29d4bc…`、
6.1 648,144 B `b4135836…`、6.6 511,856 B `1fc40216…`（比上一轮 +24K/+16K/+16K，增量即两个兼容层）。
补丁 **94,264 B**（18 文件）/ 16,613 B；`_verify109.sh` 正反向逐字节全绿（唯一差异是上游基线自带
的 `kernel/.vscode`）；新增行 `goto` 计数 0。

**记录不改**：`hook/lsm_hook.c:348/373` 的 `&head->first` 数据槽（被 `kernel_text_address()` 拒成
`-EINVAL`，钩子装不上但有日志）、branch ±128 MiB（真机布局决定）、riscv dispatcher 差异（非目标
架构）、`hooked_entries[16]` 满、KDP 无超时、`app_profile.c:246` 的 init 提权走原生 `commit_creds()`。

**真机判据**：`dmesg | grep -E 'samsung_(kdp|defex|rkp):'` 三类都应出现；修 A065 类崩溃的
tracepoint `regs` 改动见第十七轮。

## 第十九轮：最小修改 + 风格一致性审计（v1.0.9，提交 `8826fad`）

审计口径与完整数据见 `D:\root\_map\learn\FINASU-109-STYLE-AUDIT.md`。**结论：补丁在
「最小修改 / 与上游风格一致 / 整洁度」三项上已经达标**——21 文件 +2,059/−126 行全部落在
功能点，零格式噪音（新增行 CR 0、行尾空白 0、非 ASCII 0、>120 列 0、制表符只在 Kconfig），
`if (` 121/121 带空格，4 空格缩进（上游 FinaSU 自身就是 4 空格），注释 `//`:`/* */` = 75:109
（上游 427:501），版本门与 `pr_fmt` 机制与上游同款，新增行 `goto` 0。

**只改了一处**（`kernel/compat/samsung_defex.c:66`）：

```diff
-static noinline int __nocfi samsung_defex_allow(void)
+static int __nocfi noinline samsung_defex_allow(void)
```

理由是统一属性顺序：上游写 `static long __nocfi ksu_syscall_dispatch`（返回类型 → `__nocfi` →
函数名），同补丁另两处（`samsung_defex.c:235`、`samsung_kdp.c:226`）也是
`void __nocfi noinline ...`。行为不变；重编后 6.1/6.6 的 ko 字节数不变，5.15 少 64 B。

**判定不改**：4 空格缩进与 Kconfig 制表符（上游风格/语法要求，改了就是 2,059 行纯格式 diff）、
三个新 `.c` 带 SPDX（Linux coding-style 要求）、头文件不带 SPDX（与上游 include 一致）、
`(void)` 原型（比上游的 `()` 更严格）、209 行的 `branch_patch_callthru()`（拆分违反最小修改）、
`unsigned int` 与 `u32` 混用（计数 vs 指令字，与上游一致）、`noinline`（三星载体必需，上游没有）。

**验证**：三 KMI `warnings=0`、`-- KernelSU version: 33023`（621,392 / 648,144 / 511,856 B）；
补丁 **94,264 B** / 16,613 B；`_verify109.sh` 正向 == 工作树、反向 == `_pristine109`，两侧都是
逐字节相同。`_verify109.sh` 注意：`git clean -xfdq -- kernel` 会顺手删掉上游自带的
`kernel/.vscode`（被 `.gitignore` 忽略、未跟踪），从 `_pristine109/kernel/.vscode` 拷回即可。

## 第二十轮：8 字节槽替换加 call-through（修 RKP 下的自递归）

v1.0.9 新增的 `kernel/feature/mount_view_filter.c`（5.15+）与 `kernel/feature/netisolate.c`
（15-6.6 / 16-6.12）都用 `ksu_patch_text(slot, &wrapper, sizeof(wrapper), ...)` 换 syscall
表项，wrapper 内部再调 `hook->original` —— 也就是槽里的那个值。

非 RKP 内核上槽会被真写成 wrapper，`original` 是原函数，没问题；RKP 上
`ksu_patch_text()` 对 8 字节写会转成「给槽所指的函数入口写 B + 16 B 桩」，槽本身保持原函数
（`kernel/hook/arm64/patch_memory.c`）。于是 wrapper 从 `hook->original` 调回去，就是跳回自己
的分支 —— 无限递归，功能一开就爆栈。这属于我们转发语义把「干净失败」变成了「装上后崩溃」。

修法：新增 `ksu_patch_text_callthru(dst, src, len, &callthru, flags)`，语义是「装上之后，
调用者必须用 `*callthru` 去调原函数」。RKP 下直接转
`ksu_samsung_rkp_branch_patch_callthru()`（先发布 callthru 再落分支）；非 RKP 下先
`*callthru = *dst` 再走原来的 fixmap 写，失败就清回 NULL。两个调用点改成用 `hook->callthru`：
`mount_view_filter` 的 `ksu_mvf_open()`、`netisolate` 的 `ni_original()`（RKP 时返回
callthru）。还原路径仍用 `ksu_patch_text(slot, &original, ...)`——那本来就是把槽的旧值写回，
语义没有歧义；`selinux_hide`、`syscall_hook`、`lsm_hook` 同理，不动。

为什么不是只修这两处：per-consumer 修补只管今天，上游每版都可能在 `feature/` 下再加一处
（v1.0.9 一次就加了两处），人工审计必然滞后。把「安装 + 发布 callthru」收进一个 API 后，
`_check_callthru.sh` 就能静态挡住新增的裸 8 字节安装，已接进 `_verify109.sh`。业务逻辑与对外
接口都没变，`ksu_patch_text()` 的既有语义也没改。

**验证**：三 KMI `warnings=0`、`-- KernelSU version: 33023`（626,128 / 649,360 / 513,072 B）；
补丁 **102,544 B**（21 文件 +1,902/−101）/ 16,613 B；`_verify109.sh` 正向 == 工作树、反向 ==
`_pristine109`，两侧逐字节相同，门禁 `check_callthru: ok`。

## 第二十一轮：把最后三处安装也收进 call-through（提交 `6eb155a`）

第二十轮新增了 `ksu_patch_text_callthru()` 并迁了两个自递归站点，但同一套「先发布 callthru、
再落分支」的逻辑在 `selinux_hide.c` 里还手写着一遍：

- `selinux_hide_install_write_op()` 自己判 RKP、自己调
  `ksu_samsung_rkp_branch_patch_callthru()`、失败再清指针——和 API 内部完全相同；
- `hook_selinux_status_open()` 还是「先 `ksu_patch_text()` 落分支，再
  `ksu_samsung_rkp_get_stub()` 取桩写进 `callthru_sel_open_handle_status`」。RKP 上这两步之间
  读 `callthru_sel_open_handle_status` 的 `my_sel_open_handle_status()` 会拿到 NULL 并对所有进程
  （含 init、system_server）返回 `-EIO`，正是第二十轮想消灭的那类窗口，只是漏了这一处。

改法只有两行：`install_write_op()` 的整个函数体换成一次
`ksu_patch_text_callthru(slot, &hook, sizeof(hook), (void **)callthru, KSU_PATCH_TEXT_FLUSH_DCACHE)`
（失败时 API 自己清 callthru，这里只负责清 `*orig`）；`hook_selinux_status_open()` 改走同一个
API，成功后用返回的 `callthru_sel_open_handle_status` 判空——`if (callthru_sel_open_handle_status)
return 0;`，下面的 revert 分支保留成 API 回退时的兜底。净效果：`selinux_hide.c` 少了 15 行，
四个安装点全部只经一个 API。

顺带把门禁收紧：`_check_callthru.sh` 不再整体豁免 `selinux_hide.c`（只剩两处还原与一处 revert，
都是把槽的旧值写回），规则 1 的还原判定从 `*original*` 放宽到 `*orig*`，规则 2 接受
`callthru_` 前缀的全局命名。现在 `check_callthru: ok` 且没有任何主题文件被豁免。

**验证**：三 KMI `warnings=0`、`-- KernelSU version: 33023`（626,704 / 649,392 / 513,104 B）；
补丁 **102,344 B**（21 文件 +1,889/−103）/ 16,613 B；`_verify109.sh` 正向 == 工作树、反向 ==
`_pristine109`，两侧逐字节相同；新增行 goto 计数 0。

## 第二十二轮：基线同步到 v1.0.10（2026-10-25）

上游 v1.0.10 = tag `0ce48f74a5`（版本 33034）。相对 v1.0.9，上游改了很多文件，但**我们补丁
碰到的 21 个文件里只有 3 个有变化**，且都发生在切口附近：

| 文件 | 上游改了什么 | 对补丁的影响 |
|---|---|---|
| `kernel/feature/mount_view_filter.c` | 18 行：挂载视图加「root mount 直接放行」早退；`show` 回调改成 `down_read_trylock` 防锁环 | 相邻但不同位置，重放无冲突 |
| `kernel/runtime/ksud_integration.c` | 9 行：`input_hook_stopped` 从函数内 static 改成全局 `READ_ONCE/WRITE_ONCE` | 相邻但不同位置，重放无冲突 |
| `kernel/feature/selinux_hide.c` | −5 行：删掉 `struct mutex status_lock` 及其注册/反注册 | **唯一冲突**：我们的 `install_write_op()` 用该锁串行化安装 |

`selinux_hide.c` 的冲突按上游新结构收口，而不是把锁加回来：上游删锁的理由是安装只在
`ksu_selinux_hide_init()` 里发生一次，而我们的安装路径（`install_write_op()`）本来就被
`callthru` API 的「先发布指针、再落分支」保护，锁不再承担正确性职责。改法是把 hunk 里的
`mutex_lock(&status_lock)` / `mutex_unlock()` 两行直接去掉，其余不变——补丁因此比 1.0.9 版
多 14 字节（102,358 B），文件数与增删行不变。

新树 `D:\root\finasu-work\_src110`：基线提交 `1a7c474`（v1.0.10 tarball，非 git clone，
`git ls-tree` 与上游 tarball 逐字节一致），补丁提交 `2700bb7`。重放方式照旧——先
`git apply --check` 定位冲突，`--reject` 只留 `.rej` 里那一个 hunk，手工按上游结构收口，
再 `git diff` 重新生成补丁（不用编辑器改补丁文件本身）。

**验证**：`_verify110.sh` 正向 == 工作树、反向 == `_pristine110`，两侧逐字节相同，独立
`-R --check` 两个都过；`_check_callthru.sh` ok；新增行 goto 计数 0。三 KMI `warnings=0`、
`-- KernelSU version: 33034`：

| KMI | ko | sha256 |
|---|---|---|
| android13-5.15 | 627,688 B | `2946b35619ac7978…` |
| android14-6.1 | 649,320 B | `d8935c352f017bea…` |
| android15-6.6 | 513,104 B | `0275f0809c46cc20…` |


## 第二十三轮：基线同步到 v1.2.1（2026-10-26）

上游 v1.2.1 = tag `v1.2.1`（commit `be19eb90`，版本 33074），新增 `ksud: Add FinaSU Hide
bootloader state spoofing`。**`kernel/` 与 `userspace/ksud/` 里我们补丁碰到的文件逐字节未变**，
所以内核补丁 20 文件与 v1.2.0 完全相同（100,641 B）；`userspace/ksud` 的那份因为本次要加
boot-hide 接线而 +584 B（17,197 B / 3 文件 +238 −39）：

- `late_load.rs`：在 `init_event::run_stage("boot-completed", …)` 之后、`mount_guard` 清理之前
  调一次 `crate::boot_hide::boot_apply()`。late-load 跑在 post-fs-data 之后，上游在
  `init_event.rs` 的那次调用永远不会发生；补这一次之后，「启用过 FinaSU Hide」的设备每次
  late-load 都会自动重新应用 16 条属性（未启用时 `boot_apply()` 因 marker 不存在直接返回）。
- 版本串：工作树用 `Vergissmeinnicht` 分支 + detached 上游真提交 + tag `v1.2.1`，
  `KSU_VERSION_FULL` = `v1.2.1-be19eb90@Vergissmeinnicht`，功能号由 `KSU_GIT_VERSION=3074`
  钉成 33074。

验证：`_verify121.sh` 正向 == 工作树、反向 == `_pristine121`，两侧逐字节；`-R --check` 两份都过；
`_check_callthru.sh` ok；新增行 goto 0。构建见上表。

已知边界不变（RKP 16 槽、KDP fail-closed、非 arm64 落桩）。boot-hide 只影响运行期属性读取，
TEE/Keymaster 类强完整性检查不读属性，改不了。

## 第二十四轮：基线同步到 v1.3.0（2026-10-29）

上游 v1.3.0 = tag `v1.3.0`（commit `9a57ccb0500344de7ee6fc53168b441ee860ade6`，
`manager: Guide NoMount VFS setup`，版本 33082），新增 per-UID unmount mountpoints
（`KSU_IOCTL_GET_TRY_UMOUNT`，cmd 33，256 条 × 264 B 条目）、NoMount 数据面校验与恢复、
SUSFS 状态上报、root shell 解析、Manager 主题、Sui 诊断。

**内核侧漂移为零**：v1.2.1→v1.3.0 只改了 `kernel/feature/kernel_umount.c`、
`kernel/include/uapi/supercall.h`、`kernel/supercall/dispatch.c`，都不在我们补丁的 20 个文件里，
所以内核补丁仍可逐字节应用（`patch -p1 --dry-run` 20/20 check OK）。新 ioctl 不需要白名单：
`syscall_hook.c` 是对 `ksu_syscall_table` 槽位做通用 hook，不按命令号门控。

内核补丁本轮唯一的代码改动是给 `ksu_sys_read` / `ksu_sys_fstat` 补 `noinline`
（`static long __nocfi noinline ...`，`kernel/runtime/ksud_integration.c`）：它们被
`ksu_syscall_table_hook(__NR_read, ksu_sys_read, &orig_sys_read)` 按地址安装，和 delta 里
其它 `__nocfi` 包装器必须一致，否则编译器可能把函数内联掉、把地址指到别处。这两个函数体
本来就是补丁新增行，所以文件仍 20 个 / +1,807 −105，只是字节数 100,641 → 100,699。

用户态那份只有 1 个 hunk 需要重锚（`late_load.rs` hunk #5）：上游新增了
`umount_config::apply()` 步骤，并把 `mount_guard::clear_if_ours(...)` 改成可失败
（`...?`），我们 hunk 的尾部上下文行因此失配。重贴后顺序是
`umount_config::apply()` → step 11/12/13 → `boot_apply()` → `clear_if_ours(...)?` →
`Ok(())`，所以上游新引入的失败早退不会跳过 boot-hide 重放。补丁 17,197 → 17,204 B
（3 文件 / +238 −39 / 11 个 hunk）；新旧补丁的差异只有 3 处 `index` 行和这条上下文行。

版本串：`_src130` 是从 `_upstream105` 本地克隆的真 git 历史（`git rev-list --count` = 3082），
分支 `Vergissmeinnicht`，HEAD 停在 tag 上，所以 `KSU_VERSION` 自算 33082、
`KSU_VERSION_FULL` = `v1.3.0-9a57ccb0@Vergissmeinnicht`，**不再需要 marker 填充提交或
make 变量覆盖**。三个 ko 已实测含该完整串。

验证：`_verify130.sh` 正向 == 工作树、反向 == `_pristine130`，两侧逐字节（排除两个 uapi 目录）；
`-R --check` 两份都过；新增行 `goto` 0 处；三 KMI 编译 `warnings=0`，`vermagic` 与目标固件串
逐字符一致。构建见上表（本轮 ko 尺寸比 v1.2.1 各 +1.9~7.8 KB，对应新增的 umount-config 代码）。

已知边界不变（RKP 16 槽、KDP fail-closed、非 arm64 落桩）。DFRoot 专版内嵌的裁剪 ksud
仍只带 5.15 + 6.6 四个资产（6.1 上 DFRoot 升权不可用），独立 `ksud` 带全部六个。
