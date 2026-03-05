# Pragma

**A hardware optimizer. A language. An operating system.**

Pragma is an attempt to rethink how hardware, software, and firmware are expressed and work together, by making the whole stack acccessible to the people who build and run systems, not handfuls of specialists at each level.
---

## What is this?

Pragma has three parts, and they're designed to work together:

### 🔧 Metal — Hardware Optimization

Metal is a firmware optimization tool built on [coreboot](https://coreboot.org), the open-source firmware project. It uses Bayesian optimization ([BoTorch](https://botorch.org)) to systematically explore Intel FSP (Firmware Support Package) and AMD AGESA (AMD Generic Encapsulated Software Architecture) parameters — things like memory interleaving modes, UPI link speeds, prefetcher configurations, power states, and cache allocation policies — to find configurations that improve performance for your specific workload on your specific hardware.

Instead of figuring out BIOS settings from forum posts and LLM responses, Metal lets someone define safe parameter bounds, then autonomously runs benchmarks across configurations over days or weeks, learning which combinations work best. The result is empirically-discovered firmware tuning.

The companion project **[Intel-FSP-Param-Catalog](https://github.com/albazzaztariq/Intel-FSP-Param-Catalog)** provides a browsable, searchable reference of all 37,000+ FSP parameters across 18 Intel platforms — parsed directly from Intel's published header files. Though not a proper "docs", it serves right now as the best documentation
publicly available from Intel at this time.

### 📝 The Language

Pragma is also a programming language designed to express kernel-level concepts in terms that read more like intent than implementation. The idea is that you shouldn't need to swim through millions of lines of C to understand or modify how your system behaves.

The language is relaxed about the things that don't matter — delimiters, indentation rules, inlining conventions — and precise about the things that do. You can express high-level policy (
if something == somethingElse then
    x=5
    skip
else if something != somethingElse then
    x=1
else if something == nothing
FREE<buf>
// do something
end if) or drop down to hardware-level specifics ("set CHA snoop throttle to medium on socket 0") in the same program.

The goal is to shrink the gap between systems programming and application development. If you're an admin or app developer who wants to understand what's under the hood, modify scheduling behavior, or add a custom IPC mechanism, you shouldn't need to become a kernel engineer first. You should be able to read the relevant part of the kernel and understand what it says.

### 🖥️ The OS

Pragma OS is a configurable kernel that loads Linux as a driver rather than treating it as the operating system.

This is a deliberate architectural choice. Every alternative OS in history has faced the same wall: hardware compatibility. Linux has thousands of engineers and decades of work behind its driver ecosystem. Instead of reimplementing all of that (or accepting a fraction of hardware support), Pragma treats the Linux kernel as its I/O subsystem. Linux thinks it owns the hardware. Pragma sits above it and provides its own execution model, memory management, scheduling, and IPC — consuming Linux's capabilities through its existing interfaces without forking or reimplementing any of it.

The result is a thin, configurable skeleton that gives you control over things most operating systems hardwire:

**Process Management & Scheduling**
- Scheduling algorithm selection and composition (CFS, EDF, priority-based, custom)
- Per-workload scheduling policies and CPU affinity
- Core isolation and dedication strategies
- Context switch behavior and preemption granularity
- Process group hierarchies and cgroup-like resource accounting
- Real-time scheduling guarantees and deadline management

**Memory Management**
- Page size selection (4K, 2M, 1G huge pages, mixed)
- NUMA allocation policies and memory placement
- Memory overcommit behavior and OOM strategies
- Swap policies, thresholds, and backend selection
- Transparent huge page behavior per-workload
- DRAM vs CXL-attached memory tiering policies
- Memory bandwidth allocation (Intel MBA / AMD QoS)
- Page reclaim algorithms and cache pressure tuning

**Inter-Process Communication**
- IPC mechanism selection (shared memory, message queues, pipes, custom)
- Zero-copy transfer policies
- IPC namespace isolation and visibility
- Synchronization primitive selection (futex, spinlock, hybrid)
- Lock contention monitoring and adaptive strategies

**Network Stack**
- Protocol stack selection and composition
- TCP congestion control algorithm per-connection or per-workload
- Socket buffer sizing and memory allocation
- RSS (Receive Side Scaling) and RPS/RFS configuration
- XDP/eBPF program attachment points
- Network namespace configuration
- QoS classification and traffic shaping
- Firewall rule compilation strategy (nftables, iptables, or direct)
- Connection tracking table sizing and timeout policies
- Interface bonding, bridging, and VLAN configuration
- MTU and segmentation offload policies
- DNS resolution strategy and caching

**Storage & Persistence**
- Filesystem selection per-mount (ext4, XFS, btrfs, ZFS, tmpfs)
- I/O scheduler selection per-device (none, mq-deadline, BFQ, kyber)
- Readahead and writeback policies
- Journal mode and fsync behavior
- Block layer queue depth and merging
- NVMe multipath and namespace management
- Caching tiers (RAM → NVMe → spinning disk)

**Security & Isolation**
- Namespace configuration (mount, PID, network, user, IPC, cgroup, time)
- Capability sets and privilege boundaries
- Seccomp filter policies
- SELinux/AppArmor policy selection
- IOMMU and DMA protection policies
- Secure boot chain verification
- Kernel lockdown level
- Address space layout randomization (ASLR) behavior
- Stack protector and control flow integrity settings

**Power & Thermal**
- CPU frequency governor selection and parameters
- C-state and P-state policies per-core
- Package-level power limits (RAPL)
- Thermal throttling thresholds and response curves
- Device power management (PCIe ASPM, USB autosuspend)
- Workload-aware power profiles

**Device & Hardware**
- PCIe link speed and width negotiation
- Interrupt affinity and routing (MSI-X distribution)
- DMA engine allocation
- GPU compute vs display resource partitioning
- Peripheral clock gating policies
- Watchdog timer configuration
- ACPI table interpretation overrides

**Observability & Debug**
- Tracing subsystem selection (ftrace, perf, eBPF)
- Logging verbosity and routing per-subsystem
- Performance counter access policies
- Core dump behavior and storage
- Kernel live patching policies
- Audit subsystem configuration

This isn't a theoretical list — these are all real knobs that exist in Linux today, scattered across sysctl, procfs, sysfs, kernel boot parameters, module parameters, and compile-time Kconfig options. Pragma's contribution is making them navigable, composable, and expressible in one coherent language instead of six different configuration mechanisms with six different syntaxes.

---

## Why?

Because if you look at something like the Linux kernel's [`net/`](https://elixir.bootlin.com/linux/v6.19.3/source/net) directory, you'll find ~70 subdirectories at the same level — Bluetooth sitting next to bridge sitting next to IPv4 sitting next to netfilter sitting next to CAN bus. Hardware-specific code, protocol families, filtering frameworks, and utility layers all mixed together with no navigable hierarchy. Not all of them apply to any given system. There's no way to say "show me only what's relevant to *my* hardware, running *these* protocols, using *this* filtering approach" and get a clean, scoped view of the code that actually matters.

What you'd want is something like faceted navigation — pick your hardware, it narrows to applicable protocols; pick a protocol, it narrows to related tooling and configuration. A dependency-aware configuration browser instead of a flat directory listing with 30 years of accumulated everything.

That's what Pragma aims to be. Not a replacement for Linux — Linux is extraordinary at what it does — but a layer that makes the configurability that's already there actually usable by the people who need it.

---

## Status

Early stage. Metal is furthest along — the FSP parameter catalog is live, the optimization framework is being built. The language and OS are in design. We're building in public because the firmware community has always worked that way, and because the people most likely to help are the ones who already live in this space.

If you work on coreboot, Dasharo, LinuxBoot, or open server firmware, we'd love to hear from you.

---

## Repos

| Repository | Description |
|---|---|
| [Pragma-Metal](https://github.com/albazzaztariq/Pragma-Metal) | Hardware optimization tool — firmware parameter tuning via Bayesian optimization |
| [Intel-FSP-Param-Catalog](https://github.com/albazzaztariq/Intel-FSP-Param-Catalog) | Browsable reference of 37,000+ Intel FSP parameters across 18 platforms |

---

*Pragma is a project by [Tariq Albazzaz](https://github.com/albazzaztariq).*
