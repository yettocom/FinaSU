# Samsung patch set for FinaSU v1.0.9 (KDP / RKP / DEFEX)

Two patches that add Samsung Galaxy support (KNOX-style KDP credentials, RKP-safe
kernel text patching, DEFEX bypass) on top of the **FinaSU v1.0.9** tag
(`f5ad2e83`, `KSU_VERSION` 33023). They supersede the `../1.0.5/` set.

| File | Size | Scope |
| --- | --- | --- |
| `FinaSU-v1.0.9-samsung-kdp-rkp-defex.patch` | 102,344 B | 21 kernel files, +1,889 / -103 |
| `FinaSU-v1.0.9-samsung-lateload.patch` | 16,613 B | 3 ksud (Rust) files, +232 / -39 |

Apply both against a pristine v1.0.9 checkout, in any order (they touch disjoint
files). Reverse-applying is byte-exact: `git apply -R --check` passes for each, and
reverting both reproduces the pristine tree.

## What changed since the v1.0.5 set

### Rebase onto v1.0.9

v1.0.5 -> v1.0.9 changes 174 files, but only four patch targets moved:
`kernel/Kbuild`, `kernel/Kconfig`, `kernel/core/init.c` and
`kernel/feature/selinux_hide.c`. Three of those were replays rather than
conflicts. v1.0.9 stores its SUSFS integration behind
`#ifndef CONFIG_KSU_SUSFS`, so the Samsung call-through helpers now live inside
that guard (their `.c` file is only built without in-kernel SUSFS); the RKP
header stays outside it because the status-open hook uses it in both builds.

### KDP and DEFEX were never actually compiled in

`kernel/Kbuild` set only make variables (`CONFIG_KSU_SAMSUNG_KDP ?= y`,
`_DEFEX ?= y`) while the sources test preprocessor macros, and out-of-tree module
builds do not read `Kconfig`. Every ko built from the earlier sets therefore
carried KDP and DEFEX as stubs. The patches now pass
`-DCONFIG_KSU_SAMSUNG_KDP=1` and `-DCONFIG_KSU_SAMSUNG_DEFEX=1` explicitly.

Quick check on a built module:

```sh
strings kernelsu.ko | grep -c 'samsung_kdp:'      # non-zero
strings kernelsu.ko | grep -c 'samsung_defex:'    # non-zero
```

### Call-through for 8-byte slot replacement

New `ksu_patch_text_callthru()` in `kernel/hook/patch_memory.[ch]`. On arm64 it
forwards to the RKP branch patcher (which publishes the stub before the branch
lands); the generic fallback publishes the original first and only then patches.
Under RKP the grafted function gets a branch at its entry while the slot itself
keeps the original pointer, so a wrapper that calls the slot re-enters its own
replacement. `mount_view_filter.c` and `netisolate.c` did exactly that and
recurse until the stack overflows; both now call through, as do
`selinux_hide.c`, `syscall_hook.c` and `lsm_hook.c`.

### selinux_hide

The write-op install helper delegates to the new API instead of duplicating the
RKP split. A failed status-open install keeps the slot so `restore_all()` can
retry it, and the last window where `callthru_sel_open_handle_status` was still
NULL while the branch was already live is gone.

### Syscall frame

`ksu_sys_enter_handler()` reads the poisoned frame from the tracepoint argument
instead of `task_pt_regs(current)` (ported from diabl0w's `bbd9b248`; upstream
KernelSU still uses the stack-top frame).

### Earlier audit rounds retained

Restore is retried and a site whose restore fails is kept (no `vfree` of a page
that still backs a branch), a failed instruction patch is rolled back and logged,
a stub is not recycled once its call-through has been published, slot targets are
validated with `kernel_text_address()`, `kernelsu_exit()` calls
`ksu_samsung_defex_exit()` / `ksu_samsung_rkp_exit()`, every cred drop goes
through `ksu_put_cred()`, DEFEX keeps bypass working when the credential helpers
are missing, and the KDP worker compares `user_ns` as well as `user`.

## Validation

* `android13-5.15`, `android14-6.1`, `android15-6.6`: build with 0 warnings, all
  report `KernelSU version: 32997` -> rebuilt at `33023` with the v1.0.9 tree.
* Forward apply onto the pristine tag reproduces the patched tree byte for byte;
  each patch passes `git apply -R --check`; reverting both reproduces the tag.
* No `goto` in added lines; the files keep their upstream indentation and style.

## Credits

The RKP text-patch route (branch at the function entry into a 16-byte stub) comes
from [diabl0w's KernelSU fork](https://github.com/diabl0w/KernelSU). The
call-through protocol, the KDP and DEFEX layers, and the audit fixes above are
ours.
