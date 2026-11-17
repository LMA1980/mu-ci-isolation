# mu-ci-isolation

<p align="center">
  <a href="https://hook.lma1980.duckdns.org/jobs">
    <img src="https://hook.lma1980.duckdns.org/cards/LMA1980/mu-ci-isolation/card.svg" alt="mu-ci-isolation Factory Scorecard" width="520"/>
  </a>
</p>

<p align="center">
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci-isolation/status.svg" alt="CI Status"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci-isolation/score.svg" alt="Quality Score"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci-isolation/coverage.svg" alt="Coverage"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci-isolation/benchmark.svg" alt="Benchmark"/></a>
</p>

______________________________________________________________________

3-layer monotonic privilege attenuation and dual-backend sandboxing engine for the [`mu-ci`](https://github.com/LMA1980/mu-ci) platform across heterogeneous hosts (SpacemiT K3 RISC-V and Intel x86_64).

______________________________________________________________________

## Downstream Consumers

`mu-ci-isolation` provides sandbox execution and host privilege boundary enforcement across five platform daemons:

- **[`mu-ci-worker`](https://github.com/LMA1980/mu-ci-worker)**: Isolates CI stage tool invocations (`mu-ci-tools`) using `KernelSandbox` or `HypervisorSandbox`.
- **[`mu-cd-agent`](https://github.com/LMA1980/mu-cd-agent)**: Executes CAS integration-test (IT) bundle gates inside `KernelSandbox` (`MAX_IT_SANDBOX_PIDS = 256`).
- **[`mu-observable-hub`](https://github.com/LMA1980/mu-observable-hub)**: Applies monotonic isolation boundaries to control-plane query/storage execution.
- **[`mu-observable-collector`](https://github.com/LMA1980/mu-observable-collector)**: Sandboxes collector ingestion workers.
- **[`mu-observable-sentinel`](https://github.com/LMA1980/mu-observable-sentinel)**: Sandboxes edge health and anomaly sentinel probes.

______________________________________________________________________

## 3-Layer Monotonic Privilege Attenuation

Privileges are monotonically attenuated from the physical host down to individual task execution. Any request for elevated privileges beyond the enclosing layer fails closed immediately with `SecurityGrantDenied`.

```text
+--------------------------------------------------------------------------+
| Layer 1: Operator Host Ceiling (MU_ISOLATION_CEILING_PATH,               |
|          default /etc/mu-isolation/ceiling.yml)                          |
| Host-wide maximum memory, core allocations, network CIDRs, devices       |
+--------------------------------------------------------------------------+
                                    │  Monotonic Attenuation (∩)
                                    ▼
+--------------------------------------------------------------------------+
| Layer 2: Desired Profiles (.muci/security.yml,                           |
|          fallback .actionrunner/security.yml)                            |
| Repository-level boundaries: pure-offline, network-restricted, full      |
+--------------------------------------------------------------------------+
                                    │  Monotonic Attenuation (∩)
                                    ▼
+--------------------------------------------------------------------------+
| Layer 3: Task/Stage Constraints (.muci/mu-ci.yml / .mucd/mu-cd.yml,      |
|          fallback .actionrunner/ActionRunner.yml / .muconde/MuConDe.yml) |
| Granular per-step execution constraints and read/write/exec mounts       |
+--------------------------------------------------------------------------+
```

### Invariant: Monotonic Privilege Rule

$$\\text{Effective Grants} = \\text{Ceiling} \\cap \\text{Profile} \\cap \\text{Task}$$
If $\\text{Task} \\not\\subseteq \\text{Profile}$ or $\\text{Profile} \\not\\subseteq \\text{Ceiling}$, the engine refuses execution and emits `SecurityGrantDenied`.

______________________________________________________________________

## Dual-Backend Isolation Architecture

`mu-ci-isolation` abstracts execution through the `IsolationBackend` trait (`src/backend.rs`) with two specialized backends:

```rust
pub trait IsolationBackend: Send + Sync {
    /// Return the backend kind (SandboxBackendKind::Kernel or SandboxBackendKind::Hypervisor).
    fn backend_kind(&self) -> SandboxBackendKind;

    /// Validate that the host architecture, hardware constraints, and policy settings
    /// are compatible with this isolation backend.
    fn validate_host_support(&self, policy: &EffectiveSandboxPolicy) -> Result<(), IsolationError>;

    /// Lower an EffectiveSandboxPolicy and command argv into a concrete SandboxExecutionPlan.
    fn build_execution_plan(
        &self,
        policy: &EffectiveSandboxPolicy,
        argv: &[OsString],
    ) -> Result<SandboxExecutionPlan, IsolationError>;

    /// Execute or instantiate the sandboxed task from a verified SandboxExecutionPlan.
    fn spawn_isolated(
        &self,
        plan: &SandboxExecutionPlan,
    ) -> Result<IsolatedChildHandle, IsolationError>;
}
```

### 1. `KernelSandbox` (Containerless Linux Security)

- **Landlock LSM**: Unprivileged filesystem access restriction rules (`LandlockPathRule`).
- **cgroupv2 Resource Envelopes**: Enforces CPU quotas, memory ceilings, and PID limits (`max_pids`).
- **Linux Namespaces**: Unshares `CLONE_NEWNS`, `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWIPC`, `CLONE_NEWUTS`, and `CLONE_NEWUSER`.
- **`PR_SET_NO_NEW_PRIVS`**: Disallows obtaining new privileges via setuid/setgid binaries.
- **Seccomp-BPF**: Filters dangerous syscalls (`kexec`, `reboot`, raw sockets, `ptrace`).
- **SpacemiT K3 `/proc/set_ai_thread`**: Conditional thread registration prior to core affinity pinning on SpacemiT-patched kernels (`register_ai_thread_if_present`).
- **Bounded `RLIMIT_MEMLOCK` Ceiling**: Strictly enforces bounded memlock limits (`MAX_K3_MEMLOCK_KB = 29_360_128` KB / 28 GiB on a 32 GiB host), preserving ~3.3 GiB strictly for kernel/OS stability. Never sets `unlimited`.

### 2. `HypervisorSandbox` (KVM MicroVM)

- **Sub-50ms Cold Boots**: Lightweight microVM runtime providing hardware-enforced virtual machine boundaries (`MicroVmConfig`).
- **Cross-Architecture Virtualization**:
  - RISC-V Hypervisor (`H`) extension on SpacemiT K3 (`worker-riscv64`).
  - Intel VT-x hardware acceleration on Alder Lake (`node-a`) and Xeon (`node-b`).
- **Complete Guest Kernel Isolation**: Used for untrusted public PR validation and hostile execution payloads.

______________________________________________________________________

## Target Node Capabilities

| Node             | Architecture           | Cores / RAM                         | Primary Isolation Backend             | Hardware Acceleration           |
| ---------------- | ---------------------- | ----------------------------------- | ------------------------------------- | ------------------------------- |
| `node-a`         | Intel N100 (x86_64)    | 4C / 32GB DDR5                      | `KernelSandbox` / `HypervisorSandbox` | Intel VT-x                      |
| `node-b`         | Intel Xeon E3 (x86_64) | 4C/8T / 32GB DDR3                   | `KernelSandbox` / `HypervisorSandbox` | Intel VT-x                      |
| `worker-riscv64` | SpacemiT K3 (riscv64)  | 16C (8×X100 + 8×A100) / 32GB LPDDR5 | `KernelSandbox` / `HypervisorSandbox` | Landlock / RVV / RISC-V KVM `H` |

______________________________________________________________________

## License

Licensed under the GNU Lesser General Public License v3.0 with the LGPL-3.0 Linking Exception (LGPL-3.0-only WITH LGPL-3.0-linking-exception). See [LICENSE](LICENSE) for details.
