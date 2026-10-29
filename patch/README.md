# FinaSU + Samsung (KDP / RKP / DEFEX) patches

Kernel-side support for Samsung KDP, RKP and DEFEX so KernelSU/FinaSU can co-exist
with them. Based on `LyraVoid/FinaSU` tag **v1.3.2** (`9b32bde4`, version 33090).

| patch | target | size |
| --- | --- | --- |
| `FinaSU-v1.3.2-samsung-kdp-rkp-defex.patch` | `kernel/` — 20 files, +1807 −105 | 100,764 B |
| `FinaSU-v1.3.2-samsung-lateload.patch` | `userspace/ksud/src/` — 3 files, +238 −39 | 17,204 B |

Apply with:

```bash
git clone https://github.com/LyraVoid/FinaSU -b v1.3.2 && cd FinaSU
patch -p1 --forward --batch < FinaSU-v1.3.2-samsung-kdp-rkp-defex.patch
patch -p1 --forward --batch < FinaSU-v1.3.2-samsung-lateload.patch
```

## What changed for v1.3.2

- Nothing to adapt: upstream v1.3.1 → v1.3.2 touches 97 files but not one file under
  `kernel/`, and it does not touch `userspace/ksud/src/{defs,late_load,utils}.rs`
  either. Both patches are byte-identical to the v1.3.1 ones (same sha256); only the
  file names carry the new version.
- Upstream's new surface is outside the patch: `userspace/ksud/src/boot_facts.rs`
  (read-only `ksud boot-info kernel-facts`), YukiZygisk/BreZygisk provider detection,
  and manager-side UI work.
- The `kernel/Kbuild` layout and the `selinux_hide.c` rewrite from v1.3.1 are
  unchanged, so the patches still apply with no fuzz.

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
- `kernel/runtime/ksud_integration.c` — `__nocfi noinline` on the two syscall wrappers
  that are installed by address (upstream still omits `noinline`).

Both certificates the kernel trusts (`KSU_EXPECTED_*` / `*2`) are upstream features,
not part of these patches.

## Limits

- Verified at build level only (three KMIs compile with zero warnings and load
  nothing on a non-matching kernel); no on-device testing.
- RKP branch patching tracks 16 slots per boot; past that the module falls back to
  direct writes, which RKP may fault.
- `KDP` stays fail-closed: without the Samsung symbols the module loads but the
  credential path stays stock.
