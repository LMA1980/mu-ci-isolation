---
name: mu-ci-isolation
description: "Configure and enforce 3-layer monotonic privilege attenuation and KernelSandbox/HypervisorSandbox isolation with mu-ci-isolation. Use when sandboxing daemons, CI stages, or worker processes."
---

# `mu-ci-isolation` Sandboxing & Monotonic Privilege Attenuation

Use these invariants whenever integrating `mu-ci-isolation` into a daemon, worker, or test harness:

## 1. Monotonic Security Attenuation (Fail-Closed)

- Declarative security profiles (`SecurityProfile`, e.g., `.actionrunner/security.yml`) may only be **tightened** (never expanded) by workflow or task overrides (`TaskOverride`).
- Always validate overrides against the operator's `HostCeiling` before execution. Any request exceeding the host ceiling (CPU quota, memory ceiling, `max_pids`, `memlock_bytes`, or filesystem paths) **must** fail closed with `IsolationError::SecurityGrantDenied`.
- For `action-runner` service account profiles, enforce `max_pids: 256` and explicit `read_paths` / `exec_paths` (`/usr/bin`, `/usr/local/bin`, `/bin`, `/var/lib/action-runner/.cargo/bin`, `/var/lib/action-runner/.rustup`, `/var/lib/action-runner/.local/bin`).

## 2. Choosing `KernelSandbox` vs. `HypervisorSandbox`

- **`KernelSandbox` (Bare-Metal Linux Landlock V3+, `cgroupv2`, Namespaces, `RLIMIT_MEMLOCK`)**:
  - Use when zero-jitter hardware performance counters or SpacemiT K3 Cluster 1 (`A100`, cores `8–15`) `/proc/set_ai_thread` kernel registration is required, or for self-confining edge daemons at startup.
- **`HypervisorSandbox` (`/dev/kvm` Hardware Virtualization)**:
  - Use when strong guest-kernel isolation is required (supported across RISC-V `H` extension on SpacemiT K3 Cluster 0 `X100` cores `0–7` and `x86_64` VT-x nodes).

## 3. Bounded `RLIMIT_MEMLOCK` Ceiling

- **Never** set `RLIMIT_MEMLOCK` (`memlock_bytes`) to `unlimited` (`RLIM_INFINITY`), as locking 100% of physical RAM starves kernel slab allocation and SSH responsiveness. Always enforce a bounded byte limit with safe OS headroom.
