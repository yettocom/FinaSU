# Samsung patches for FinaSU

Patches that make [FinaSU](https://github.com/LyraVoid/FinaSU) work on Samsung
devices whose kernels carry RKP, KDP, DEFEX and SELinux hooks. Based on tag
**v1.5.0** (`fbf8e003`, version 33174).

| Patch | Touches | Size |
|---|---|---|
| `FinaSU-v1.5.0-samsung-kdp-rkp-defex.patch` | `kernel/` - 19 files, +1820 -89 | 97,322 B |
| `FinaSU-v1.5.0-samsung-lateload.patch` | `userspace/ksud/src/` - 3 files, +241 -37 | 17,314 B |

## Apply

```sh
git clone https://github.com/LyraVoid/FinaSU -b v1.5.0 && cd FinaSU
mkdir -p kernel/compat
patch -p1 --forward --batch < FinaSU-v1.5.0-samsung-kdp-rkp-defex.patch
patch -p1 --forward --batch < FinaSU-v1.5.0-samsung-lateload.patch
```

`mkdir -p kernel/compat` is required: v1.5.0 does not ship that directory, `git
apply` creates it but `patch -p1` does not, and without it the kernel patch stops
at `patch: cannot write to file 'kernel/compat/samsung_defex.c'`.

`git apply` also works on Linux and macOS. On Windows, replace the two `uapi`
symlinks with real directories first (`kernel/include/uapi`,
`manager/app/src/main/cpp/uapi`), otherwise the DDK build cannot find
`uapi/app_profile.h`.

## What changed for v1.5.0

- Upstream moved six source files between v1.4.3 and v1.5.0: four under
  `kernel/selinux/` plus `kernel/feature/selinux_hide.c`, and two under
  `userspace/ksud/src/` (`module.rs`, `feature.rs`). Only
  `kernel/feature/selinux_hide.c` is inside our patch surface, and none of the
  hunks overlap the upstream edits, so both patches apply with zero fuzz and
  zero rejects.
- The kernel patch differs from the v1.4.3 one by a single `index` line on
  `kernel/feature/selinux_hide.c`; the userspace patch is byte-identical.
- Upstream added SELinux policy-generation tracking (`ksu_set_original_types()`,
  `ksu_advance_original_types()`, `ksu_type_is_original()`, and a `seqno`
  argument on `reset_avc_cache()`). That work is orthogonal to the RKP layer:
  none of it touches `ksu_patch_text()` or the branch patcher, and our code
  calls none of the new symbols.
- Verified end to end: `git archive v1.5.0` plus these two patches reproduces
  the maintainer's tree byte for byte (1,897 files, sha256 compared).

## What the patches do

- `kernel/compat/samsung_rkp.c` - branch patcher for kernels whose text is
  protected from EL2. Every 8-byte text write is turned into a branch at the
  callee entry plus a 16-byte stub that replays the displaced instruction and
  jumps back. Call-through entries are published before the branch is reachable.
- `kernel/compat/samsung_kdp.c` - routes credential commits through Samsung's
  `kdp_usecount_*_and_test()` accounting when KDP is present, and falls back to
  plain `commit_creds()` when it is not.
- `kernel/compat/samsung_defex.c` - DEFEX enforcement is muted per TASK (not
  globally) while a module is being loaded, then restored.
- `kernel/feature/selinux_hide.c`, `kernel/hook/lsm_hook.c`,
  `kernel/hook/arm64/*` - the upstream call sites are retargeted at the branch
  patcher, and every consumer checks its call-through before calling it.
- `userspace/ksud/src/{late_load,utils,defs}.rs` - late-load stages the daemon
  before loading the module, re-runs the FinaSU Hide pass afterwards, and does
  not restart the manager.

## Limitations

- Only the three KMI this project builds are exercised: `android13-5.15`,
  `android14-6.1`, `android15-6.6` (S9180 / S9280 / S9380 firmware). Other KMIs
  need their own DDK image; the kernel module is not built for them.
- Loading the module on a non-Samsung kernel is not an error: the compat layer
  reports "not present" and the standard credential path is used.
- `CONFIG_KALLSYMS_ALL=y` is required for built-in builds (upstream
  requirement); the LKM path is unaffected.
- Never pass `vivo_module_filter` on Samsung: that hook patches
  `__arm64_sys_init_module`, the same entry DFRoot's kprobe owns.
- DFRoot is a privilege tool, not a manager. It loads the module and hands over;
  the SU manager is a separate APK, and the kernel decides manager identity by
  APK signature, not by which app launched the loader.
