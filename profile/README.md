# **Hardware Optimization - Language - Operating System**
---
This is an attempt at making things as deeply configurable as possible.  I'm trying to take hardware optimization, a language for both high-level apps and low-level systems, and an operating system with more knobs that ever easily accessible, and put it into one single solution.  Way beyond overclocking, language that easily converts from and to and is able to be expressed well in each companion language, and an operating system that has userspace apps for changing IPC messaging methodology, process scheduling, I/O handling, network stack, and other things.  Everything linux can do and BSD and others.  All rolled up into one thing that you don't have to hassle an LLM about every time you want to change something.

This all started with Claude Code and me getting back my interests in computers.  I left it when I was a kid but there were a few years from around 11-15 where I got every Sam's, O'Reilly's and other books I could grab from the bookstore.  I pretty much lived on the computer from day to night, stumbling through Linux, reading content on computing, and playing with tools I probably had no business playing with in the seedier days of pockets of backpage communities on IRC, telnet, and the like.  I never got much deeper than building simple programs and more than anything just editing scripts or programs I downloaded, most of the time even then just building binaries.  Later in college I got a formal introduction to programming and had no real issue with the class workloads. But I ended up leaving for years, just doing things at work from time to time.  I used it, so I didn't lose it, but I def didn't use it all the time.  

Then Claude Code came around.  I had seen the chatbots but hadn't really messed much with a coding agent that could write files on disk, run tests, whip up code without copypasta games with chatbots, etc.  I've basically been hooked since then.  There's not a day right now that I'm not on the computer and months have gone by.  

This project is me making an attempt at something bigger and also giving myself a formal, structured, and very large workload that I can chip away at.  The tools available even beyond LLM's is just so much larger in scope than long ago.  I can find UC Berkeley lectures on operating systems - the people who made BSD. I can go to interactive webpages that are essentially free books with exercises, and there's many books with just simple explanations followed by code snippets, instead of huge bibles big enough to be weapons. There's also just a lot more libraries out there to do anything you can imagine, from python's ML stuff to this very website where I can find thousands and thousands of projects with docs and faces and not have to worry so much about seeing a terminal window open and close quickly alongside the new program I thought was going to be a fun experience. Instead of spending hours tying together sparse documentation and getting dunked on in an IRC channel by guys twice my age, I get to take it easy and have tons of resources and my own personal assistant to even mend it all together automatically.

So that's the how I got here from me. The rest of this is from Claude with me doing some light checks. If he's not the poet you had hoped, I can't say much - it's the wizard I need.

---
Pragma has three parts, and they're designed to work together:

### 🔧 Metal — Hardware Optimization

Metal is a firmware optimization tool built on [coreboot](https://coreboot.org), the open-source firmware project. It uses Bayesian optimization ([BoTorch](https://botorch.org)) to systematically explore Intel FSP (Firmware Support Package) and AMD AGESA (AMD Generic Encapsulated Software Architecture) parameters — things like memory interleaving modes, UPI link speeds, prefetcher configurations, power states, and cache allocation policies — to find configurations that improve performance for your specific workload on your specific hardware.

Instead of figuring out BIOS settings from forum posts and LLM responses, Metal lets someone define safe parameter bounds, then autonomously runs benchmarks across configurations over days or weeks, learning which combinations work best. The result is empirically-discovered firmware tuning.

The companion project **[Firmware-Param-Catalog](https://github.com/albazzaztariq/Firmware-Param-Catalog)** provides a browsable, searchable reference of all 39,000+ FSP and AGESA parameters across 29 Intel/AMD platforms — parsed directly from published header files. Though not a proper "docs", it serves right now as the best documentation publicly available from vendors at this time.

### 📝 The Language

Pragma is also a programming language designed to express kernel-level concepts in terms that read more like intent than implementation. The idea is that you shouldn't need to swim through millions of lines of C to understand or modify how your system behaves.

The language is relaxed about the things that don't matter — delimiters, indentation rules, inlining conventions — and precise about the things that do. Standard and Base Pragma are the same language; Base just adds memory and hardware primitives. You can write readable policy logic and hardware register writes in the same file:

```
// standard — readable policy logic
constant int SNOOP_FULL    = 0
constant int SNOOP_MEDIUM  = 1
constant int SNOOP_REDUCED = 2

function throttle_for(int temp_c) returns int
  if temp_c < 70
    return SNOOP_FULL
  else if temp_c < 85
    return SNOOP_MEDIUM
  end if
  return SNOOP_REDUCED
end function

// base — write result directly to hardware
function apply(int level, int socket) returns void
  int64 base = 0xFED10000
  int64 reg  = base + (socket * 0x1000) + 0x80   // CHA snoop ctrl offset
  mem<reg>   = level
end function

function main()
  int socket = 0
  int temp   = 82              // degrees C — read from MSR in practice
  int level  = throttle_for(temp)
  apply(level, socket)
end function
```

The goal is to shrink the gap between systems programming and application development. If you're an admin or app developer who wants to understand what's under the hood, modify scheduling behavior, or add a custom IPC mechanism, you shouldn't need to become a kernel engineer first. You should be able to read the relevant part of the kernel and understand what it says.

### 🖥️ The OS

Pragma OS is a configurable kernel that loads Linux as a driver rather than treating it as the operating system.

This is a deliberate architectural choice. Every alternative OS in history has faced the same wall: hardware compatibility. Linux has thousands of engineers and decades of work behind its driver ecosystem. Instead of reimplementing all of that (or accepting a fraction of hardware support), Pragma treats the Linux kernel as its I/O subsystem. Linux thinks it owns the hardware. Pragma sits above it and provides its own execution model, memory management, scheduling, and IPC — consuming Linux's capabilities through its existing interfaces without forking or reimplementing any of it.

The result is a thin, configurable skeleton that gives you unified, easy-access control over settings that Linux today offers:

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

These are all real knobs that exist in Linux today, scattered across sysctl, procfs, sysfs, kernel boot parameters, module parameters, and compile-time Kconfig options. Pragma's contribution is making them navigable, composable, and expressible in one coherent language instead of six different configuration mechanisms with six different syntaxes. 

However, Linux can't hot-swap its own scheduler or memory manager while running without kexec or a reboot in most cases. Some things are runtime-tunable (sysctl), some require a reboot (boot parameters), some require recompilation (Kconfig). Pragma's value would be making ALL of them runtime-configurable through one interface and one language, 

---

## Why?

Because if you look at something like the Linux kernel's [`net/`](https://elixir.bootlin.com/linux/v6.19.3/source/net) directory, you'll find ~70 subdirectories at the same level — Bluetooth sitting next to bridge sitting next to IPv4 sitting next to netfilter sitting next to CAN bus. Hardware-specific code, protocol families, filtering frameworks, and utility layers all mixed together with no navigable hierarchy. Not all of them apply to any given system. There's no way to say "show me only what's relevant to *my* hardware, running *these* protocols, using *this* filtering approach" and get a clean, scoped view of the code that actually matters.

What you'd want is something like faceted navigation — pick your hardware, it narrows to applicable protocols; pick a protocol, it narrows to related tooling and configuration. A dependency-aware configuration browser instead of a flat directory listing with 30 years of accumulated everything.

That's what Pragma aims to be. Not a replacement for Linux — Linux is extraordinary at what it does — but a layer that makes the configurability that's already there actually usable by the people who need it.

---

## Status

Early stage. Metal is furthest along — the FSP and AGESA parameter catalog is live, the optimization framework and language are being built. The OS is in design. We're building in public because the people most likely to help are the ones already living in these spaces.

If you work on coreboot, Dasharo, LinuxBoot, or open server firmware, we'd love to hear from you.

---

## Repos

| Repository | Description |
|---|---|
| [Pragma-Metal](https://github.com/albazzaztariq/Pragma-Metal) | Hardware optimization tool — firmware parameter tuning via Bayesian optimization |
| [Firmware-Param-Catalog](https://github.com/albazzaztariq/Firmware-Param-Catalog) | Browsable reference of 39,000+ Intel FSP and AMD AGESA parameters across 29 platforms |

---
