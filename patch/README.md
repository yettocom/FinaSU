# FinaSU Samsung patches

Two patches that add Samsung (KDP / RKP / DEFEX) support to
[LyraVoid/FinaSU](https://github.com/LyraVoid/FinaSU). They are applied on top of
a clean checkout of the upstream tag and are not part of upstream.

| patch | files | touches |
| --- | --- | --- |
| `FinaSU-v1.3.0-samsung-kdp-rkp-defex.patch` | 20 (+1,807 −105) | `kernel/` |
| `FinaSU-v1.3.0-samsung-lateload.patch` | 3 (+238 −39) | `userspace/ksud/` |

## What this update adapts

Baseline moved from v1.2.1 (33074) to **v1.3.0 (33082, `9a57ccb0`)**.

- **Kernel patch** — content unchanged. Upstream only touched
  `kernel/feature/kernel_umount.c`, `kernel/include/uapi/supercall.h` and
  `kernel/supercall/dispatch.c` between the two tags, and none of those files is
  in this patch. The single difference is two `noinline` additions on
  `ksu_sys_read()` / `ksu_sys_fstat()`, which are installed into the syscall
  table by address.
- **ksud patch** — only hunk #5 of `late_load.rs` needed re-anchoring: upstream
  made `mount_guard::clear_if_ours(...)` fallible with `?` and added the
  `umount_config::apply()` step above it. The boot-hide re-apply still runs
  before that call, so an early failure cannot skip it.
- **No ABI change** — the new `KSU_IOCTL_GET_TRY_UMOUNT` rides the existing
  syscall hook, which is not gated per command.
- **Version strings** — read from the real upstream history, so a build reports
  `33082` and `v1.3.0-9a57ccb0@<branch>` without extra marker commits.

## Apply

```sh
git clone --depth=1 -b v1.3.0 https://github.com/LyraVoid/FinaSU.git
cd FinaSU
git apply /path/to/FinaSU-v1.3.0-samsung-kdp-rkp-defex.patch
git apply /path/to/FinaSU-v1.3.0-samsung-lateload.patch
```

`patch -p1` works as well.

## What the patches add

- `kernel/compat/samsung_rkp.c` — routes RKP-protected branch sites through a
  stub instead of writing text in place.
- `kernel/compat/samsung_kdp.c` — swaps the credential pair KDP tracks, so
  `commit_creds()` stays consistent with KDP's reference counting.
- `kernel/compat/samsung_defex.c` — opens a 15 s DEFEX window for the module
  load and restores the entry afterwards.
- `kernel/feature/selinux_hide.c`, `kernel/hook/lsm_hook.c`,
  `kernel/hook/arm64/syscall_hook.c` — call-through variants, so a patched slot
  never calls itself.
- `userspace/ksud/src/late_load.rs` — re-applies the boot-hide stage after
  post-fs-data and drops the automatic manager restart.

## Limits

- Builds for android13-5.15, android14-6.1 and android15-6.6 with no warnings.
  Device validation is out of scope for this repository.
- Non-Samsung kernels still build: the Samsung entry points return 0 when their
  symbols are missing.
