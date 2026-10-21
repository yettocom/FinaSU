# Samsung patch set for FinaSU v1.0.5 (KDP / RKP / DEFEX)

Two patches that add Samsung Galaxy support (KNOX-style KDP credentials, RKP-safe
kernel text patching, DEFEX bypass) on top of the **FinaSU v1.0.5** tag
(`d015fbc1`, `KSU_VERSION` 32990). They supersede the `../1.0.4/` set.

| File | Size | Scope |
| --- | --- | --- |
| `FinaSU-v1.0.5-samsung-kdp-rkp-defex.patch` | 88,848 B | 17 kernel files, +1,751 / -77 |
| `FinaSU-v1.0.5-samsung-lateload.patch` | 16,619 B | 3 ksud (Rust) files, +232 / -39 |

Apply both against a pristine v1.0.5 checkout, in any order (they touch disjoint
files). Reverse-applying is byte-exact: `git apply -R --check` passes for each, and
reverting both reproduces the pristine tree.

## What changed since the v1.0.4 set

### Rebase onto v1.0.5

Upstream v1.0.4 -> v1.0.5 changes 104 files, but only one of them
(`kernel/runtime/ksud_integration.c`, the `is_init_rc()` simplification) is
touched by the patch set, so the v1.0.5 patches apply directly with a -4 line
offset in that one hunk. The other 16 kernel files are byte-identical to v1.0.4
sources. On top of the rebase, four audit rounds landed real fixes:

### RKP (`kernel/compat/samsung_rkp.c`)

- Module exit no longer frees the stub page while branches are still installed:
  each site restore is retried once; on failure the site is kept, the count is
  not decremented, and an explicit `pr_err` warns that unloading would leave a
  jump into freed text.
- Instruction-patch failures roll back the original word(s) and free the stub
  slot (previously the slot leaked and the call-through pointer was left
  dangling).
- Slot targets are validated with `kernel_text_address()` (resolved optionally;
  missing symbol only downgrades to an `pr_info`, never to a hard failure).
- `branch_patch_callthru` / `get_stub` re-check `ksu_rkp_active` under the lock;
  writes to `ksu_rkp_active` use `WRITE_ONCE`.
- Install and restore use one bulk `aarch64_insn_patch_text()` call (two
  entries) so no CPU observes a half-installed state.

### call-through sequencing (`kernel/feature/selinux_hide.c`, `kernel/hook/arm64/syscall_hook.c`)

- The `context` / `access` write-op hooks now publish the call-through stub
  *before* the branch lands (via `ksu_samsung_rkp_branch_patch_callthru`),
  closing the window where a caller could hit the hooked function with no valid
  original to call and get a transient `-EIO`.
- `sel_open_handle_status` gained the same treatment and now returns its status;
  the enable path aborts cleanly when any hook fails, and re-arming is
  self-healing (a failed previous restore is detected and healed before
  installing, otherwise the hook would treat its own replacement as the
  original and self-recurse).
- `syscall_hook` no longer aliases its out-parameter: the stub address handed
  back to the caller and the true original recorded in `hooked_entries[]` are
  now separate; entries are only registered after a successful patch.
- `sucompat` no longer continues the execve when the root-profile escape fails
  (previously a failed escape still ran the exec and installed a session fd).

### KDP (`kernel/compat/samsung_kdp.c`)

- Version gates corrected from 5.11 to 5.14 (`cred->ucounts` and
  `inc/dec_rlimit_ucounts` only exist from 5.14; 5.11–5.13 trees would not
  compile).
- `kdp_available` is published with `smp_store_release` and read with
  `smp_load_acquire`; the use-count function pointers gained NULL fallbacks.
- `cap_issubset` direction fixed to match the stock `cred_cap_issubset`
  semantics; `user_changed` now also compares `user_ns`.

### DEFEX (`kernel/compat/samsung_defex.c`, `kernel/selinux/selinux.c`)

- Kprobes are registered before the credential helpers are resolved: when
  `get_task_creds`/`set_task_creds` are missing, the bypass keeps working and
  only the credential synchronization is disabled (previously both were lost).
- `defex_available` uses release/acquire and readers NULL-check the helper
  pointers; `cred_flags` is zero-initialized; the no-op detection reads at most
  one word past a mismatch (bounded reads, no cross-symbol reads); the boot log
  reports the real hook state instead of always claiming "enabled".
- `is_sid_match()` refuses the allocating slow path in atomic context (the
  DEFEX kprobe runs with IRQs disabled), so the window before `cache_sid()`
  runs can no longer sleep in the probe handler.

### Build system (`kernel/Kbuild`)

- `CONFIG_KSU_SAMSUNG_KDP` / `CONFIG_KSU_SAMSUNG_DEFEX` default to `y` for
  arm64 out-of-tree builds. Out-of-tree builds do not read Kconfig, so the
  default entry point used to silently compile both features into empty stubs.

### ksud late-load (`userspace/ksud/src/{defs.rs,late_load.rs,utils.rs}`)

- Staging no longer destroys the running daemon: a failed verification of
  `/data/local/tmp/.ksud-stage` falls back to `stage_daemon()` instead of
  deleting the installed daemon and bailing (which permanently lost root).
- Daemon replacement is atomic (copy to a same-directory temp file, then
  rename): no in-place truncation, no `ETXTBSY` while the daemon runs,
  immutable handling on `EPERM`, an `EXDEV` copy fallback, and unknown errors
  are surfaced with context instead of being swallowed.
- Dropping `daemonize()` no longer loses `setsid()`/`switch_cgroups()`; the
  staged path moved into `defs.rs` as `STAGED_DAEMON_PATH`.

### Explicitly unchanged (intentional)

- The `-EIO` guards on empty call-through pointers, and the `setprocattr`
  publish window: both are structural consequences of upstream
  `ksu_lsm_hook()` installing the slot before filling `hook->original`.
- KDP commit waits without a timeout (a timeout would require transferring
  `rw_cred` ownership on the failure path, a protocol change, not a fix).
- No `goto` in any code added by these patches; existing upstream `goto`s are
  untouched.

## Verification

- Reverse apply: `git apply -R --check` clean for both patches; reverting both
  reproduces the pristine v1.0.5 tree byte-for-byte.
- Builds: `android13-5.15`, `android14-6.1`, `android15-6.6` (DDK
  `ghcr.io/ylarod/ddk-min`, `CONFIG_KSU=m CONFIG_KSU_SAMSUNG_KDP=y
  CONFIG_KSU_SAMSUNG_DEFEX=y`) -- zero warnings on all three; `cargo check
  --target aarch64-linux-android` clean for ksud.
- Real-device validation (S23 Ultra, `S9180ZCS8FZI1`, 5.15.189) is the
  remaining step; the patch set has not yet been flashed on hardware.

## Credits

The RKP text-patching route (entry branch + 16-byte stub) originates from
diabl0w's KernelSU fork; see `../1.0.4/README.md` for the full attribution.
The call-through stub publication and everything listed under "What changed"
are this fork's additions.
