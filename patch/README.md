# Samsung patches for FinaSU

Upstream FinaSU has no Samsung support. On a Samsung GKI kernel `ksu_patch_text()`
fixmap writes are rejected by EL2 (RKP / proca), credential accounting sits behind
KDP write protection, and DEFEX treats the KSU domain as a privilege-escalation
target, so the stock module either fails to install its hooks or kills the
`su` process mid-`escape_to_root`.

This directory carries the patch set that makes FinaSU work on those kernels.

Only the newest set is kept: a new release replaces the previous one. History for
each file stays in `git log -- <path>`, so older sets are still retrievable.

## Files

| Patch | Base | Size | Files | Content |
| --- | --- | --- | --- | --- |
| `FinaSU-v1.0.10-samsung-kdp-rkp-defex.patch` | FinaSU **v1.0.10** (tag `0ce48f74a5`, version 33034) | 100,641 B | 20 | kernel side: RKP branch patcher, KDP credential commit, DEFEX synchronisation, LSM / syscall / selinux_hide wiring |
| `FinaSU-v1.0.10-samsung-lateload.patch` | same | 16,613 B | 3 | `userspace/ksud`: stage the daemon before loading the module in late-load mode |

Both patches touch disjoint files, so either can be applied on its own, and both
reverse cleanly with `git apply -R` — that round trip is how they are verified
against the pristine tree.

## Apply

```sh
git -C <finasu-src> checkout 0ce48f74a5
git -c core.autocrlf=false apply FinaSU-v1.0.10-samsung-kdp-rkp-defex.patch
git -c core.autocrlf=false apply FinaSU-v1.0.10-samsung-lateload.patch
```

Build the module out of tree as usual (`make -C <kernel> M=$PWD/kernel`), with
`KSU_GIT_VERSION` pinned if you want the reported version to match the release.

## What the kernel patch does

- **RKP branch patcher** (`kernel/compat/samsung_rkp.c`): RKP blocks text writes,
  so the patcher branches a function entry to a 16-byte stub page and hands the
  caller a call-through pointer for the displaced instructions. Hook install
  must publish the call-through *before* the branch lands, otherwise the wrapper
  calls the function that was just branched and recurses until the stack dies.
- **Call-through API** (`ksu_patch_text_callthru()` in `kernel/hook/arm64/patch_memory.c`):
  one entry point so a feature never has to guess whether the slot itself or the
  branch patcher owns the original.
- **KDP** (`kernel/compat/samsung_kdp.c`): commits credentials through
  `prepare_ro_creds()` / `kdp_assign_pgd()` while the real credential bookkeeping
  runs under UMH, with a static work item and no stack lifetime games.
- **DEFEX** (`kernel/compat/samsung_defex.c`): kprobes on `task_defex_enforce` and
  `task_defex_user_exec` plus credential-triple synchronisation, so processes the
  manager patches are not re-flagged by DEFEX.
- **selinux_hide, mount_view_filter, netisolate, syscall hook, LSM hook**: routed
  through the call-through API so a hook is never installed while its original
  target is unavailable.

KDP and DEFEX are compiled unconditionally and fall back at runtime
(`kdp_available = false`) when the kernel does not export the symbols they need;
there is nothing to toggle at build time.

## Verify

```sh
git apply --check -R FinaSU-v1.0.10-samsung-*.patch   # against the patched tree
```

The release these patches were built for reports `KernelSU version: 33034` and
compiles warning-free for `android13-5.15`, `android14-6.1` and `android15-6.6`
with the DDK images `20260828`.
