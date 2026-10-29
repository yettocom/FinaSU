# FinaSU + Samsung (KDP / RKP / DEFEX) patches

Kernel-side support for Samsung KDP, RKP and DEFEX so KernelSU/FinaSU can co-exist
with them. Based on `LyraVoid/FinaSU` tag **v1.3.1** (`2a559ff4`, version 33084).

| patch | target | size |
| --- | --- | --- |
| `FinaSU-v1.3.1-samsung-kdp-rkp-defex.patch` | `kernel/` — 20 files, +1807 −105 | 100,764 B |
| `FinaSU-v1.3.1-samsung-lateload.patch` | `userspace/ksud/src/` — 3 files, +238 −39 | 17,204 B |

Apply with:

```bash
git clone https://github.com/LyraVoid/FinaSU -b v1.3.1 && cd FinaSU
patch -p1 --forward --batch < FinaSU-v1.3.1-samsung-kdp-rkp-defex.patch
patch -p1 --forward --batch < FinaSU-v1.3.1-samsung-lateload.patch
```

## What changed for v1.3.1

- Kernel patch re-anchored: upstream added a six-line `CONFIG_KALLSYMS_ALL` gate at
  the top of `kernel/Kbuild`, which moved our `kernelsu-objs +=` block down. The
  patch now applies with no fuzz.
- `kernel/feature/selinux_hide.c` was rewritten upstream (+73 −66, dual policy
  validation, `orig_valid` gate, MLS tail refilled from the source string). Both
  hunks sit after ours, so they do not overlap — our patch only routes the four
  entry points through `ksu_patch_text_callthru()` and never touches `scontext`.
- The ksud patch is byte-identical to the v1.3.0 one: upstream did not change
  `defs.rs`, `late_load.rs` or `utils.rs` between the two tags.
- `ksu_sys_read` / `ksu_sys_fstat` in `kernel/runtime/ksud_integration.c` keep the
  `__nocfi noinline` we added; upstream still installs them by address without it.

## What the patches contain

- `kernel/compat/samsung_kdp.c` — KDP credential tracking (`prepare_ro_creds`,
  `kdp_assign_pgd`, `__put_cred` accounted exactly once).
- `kernel/compat/samsung_defex.c` — DEFEX policy sync for the credentials we create.
- `kernel/compat/samsung_rkp.c` — RKP-aware text patching: every hooked entry goes
  through a call-through stub instead of a direct branch, so RKP never sees a
  rewritten instruction stream.
- `kernel/feature/selinux_hide.c` — the same call-through treatment for the SELinux
  side, plus the `setprocattr` / `sel_open_handle_status` entry points.
- `userspace/ksud` — late-load path: applies FinaSU Hide, keeps the manager alive
  instead of restarting it, staged daemon install with no partial file visible.

Both certificates the kernel trusts (`KSU_EXPECTED_*` / `*2`) are upstream features,
not part of these patches.

## Limits

Static only — nothing here was flashed to a device by the maintainer of this fork. The
call-through route assumes the Samsung RKP branch-patching primitives are available;
without them the module falls back to direct writes.
