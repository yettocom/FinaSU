# Samsung patches for FinaSU

Patches that make [FinaSU](https://github.com/LyraVoid/FinaSU) work on Samsung
devices whose kernels carry RKP, KDP, DEFEX and SELinux hooks. Based on tag
**v1.4.2** (`96104eac`, version 33155), upstream `kernel/` untouched by us.

| Patch | Touches | Size |
|---|---|---|
| `FinaSU-v1.4.2-samsung-kdp-rkp-defex.patch` | `kernel/` - 19 files, +1820 -89 | 97,322 B |
| `FinaSU-v1.4.2-samsung-lateload.patch` | `userspace/ksud/src/` - 3 files, +241 -37 | 17,314 B |

## Apply

```sh
git clone https://github.com/LyraVoid/FinaSU -b v1.4.2 && cd FinaSU
patch -p1 --forward --batch < FinaSU-v1.4.2-samsung-kdp-rkp-defex.patch
patch -p1 --forward --batch < FinaSU-v1.4.2-samsung-lateload.patch
```

`git apply` also works on Linux and macOS. On Windows, replace the two `uapi`
symlinks with real directories first (`kernel/include/uapi`,
`manager/app/src/main/cpp/uapi`), otherwise the DDK build cannot find
`uapi/app_profile.h`.

## What changed for v1.4.2

- Nothing upstream in our surface: v1.4.1 to v1.4.2 is a single commit that
  reverts the FolkZygisk HyperOS runtime compatibility work (27 files under
  `userspace/folkzygisk/`); `kernel/` is untouched. Both patches are
  byte-identical to the v1.4.1 ones.
- The kernel-side content is the v1.4.0 adaptation: re-anchored on the SUSFS
  Inline backend and on the vivo module filter, with KDP-aware credential
  commits in the two init/adb root paths.
- Late-load now fails closed: an LSM hook that cannot obtain a call-through is
  rolled back and reported instead of silently returning `-EIO` forever.

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
  `android14-6.1`, `android15-6.6` (S9180 / S9280 / S9380 firmware).
- Loading the module on a non-Samsung kernel is not an error: the compat layer
  reports "not present" and the standard credential path is used.
- `CONFIG_KALLSYMS_ALL=y` is required for built-in builds (upstream
  requirement); the LKM path is unaffected.
- Never pass `vivo_module_filter` on Samsung: that hook patches
  `__arm64_sys_init_module`, the same entry DFRoot's kprobe owns.
