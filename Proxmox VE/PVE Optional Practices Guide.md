## Proxmox VE Optional Practices Guide
### Purpose and Scope

This guide outlines **optional practices** suggested by SOClogix for operating Proxmox VE in production environments. The recommendations are intended to serve as reference points for security hygiene, performance stability, and common operational decisions—not as mandatory standards or universally “correct” configurations.

The guidance provided here is intentionally selective and experience-driven. It does not represent a CIS benchmark, formal hardening standard, or compliance requirement, nor does it attempt to catalog every possible configuration or control. Instead, it highlights approaches that may be beneficial in certain environments and deployment scenarios, depending on organizational context and priorities.

Adoption of any recommendation in this guide is discretionary. Variations, alternative approaches, or omissions are acceptable—and often appropriate—when driven by workload requirements, architectural constraints, risk tolerance, or organizational policy. Any such decisions should be made deliberately and documented accordingly.

#### Out of Scope

This guide does **not** attempt to define, replace, or compete with industry **best practices**, generalized recommendations, or standardized configuration baselines intended to apply broadly across most environments.

#### Keep in Mind

- Sections of this guide are not ordered by priority or implementation sequence.
- Each recommendation should be evaluated independently before adoption.
- Inclusion in this guide does **not** imply that a practice is required, recommended for all environments, or representative of a standard baseline.
- Do not implement an action solely because it is listed here. Always confirm alignment with organizational needs, internal policies, risk assessments, and operational constraints before proceeding.

### I. Before You Start
Prior to going through this guide it is recommended that you review the Best Practices Guide, performing any and all actions suggested there unless otherwise specified or required by your organization or the specific environment.

### II. Host-Level Security Operational Practices
This section describes **optional security hygiene measures** that are commonly considered for Proxmox VE hosts. These items are not mandatory and should be evaluated individually; implementation is recommended only when they align with organizational requirements, risk tolerance, and documented operational decisions.


#### 1. Configure kernel panic auto-reboot
It is more on the rare side to experience a kernel panic. But they do happen. A host that has a kernel panic and is stuck is one of the worst things that can happen, especially if you are away from home and you can’t “poke it in the eye.”

**This allows you to configure it to autoreboot:**
```
echo "kernel.panic = 10" | tee /etc/sysctl.d/99-kernelpanic.conf
```
```
echo "kernel.panic_on_oops = 1" | tee -a /etc/sysctl.d/99-kernelpanic.conf
```
```
sysctl -p /etc/sysctl.d/99-kernelpanic.conf
```

**Verify it applied:**
```
sysctl kernel.panic kernel.panic_on_oops
```


#### 2. Improve entropy generation with haveged

Headless servers and virtualized environments can suffer from low entropy. On Linux systems like Proxmox, entropy provides the randomness required for cryptographic operations such as key generation, TLS initialization, SSH, and other security-sensitive tasks.

Entropy is normally collected from unpredictable hardware and timing events. On systems with limited hardware randomness or no user interaction, the entropy pool may fill slowly, leading to delays or blocking behavior.

`haveged` (Hardware Volatile Entropy Gathering and Expansion) is a userspace daemon that supplements system entropy by leveraging small variations in CPU execution timing and cache behavior. This helps keep the kernel’s random number generator sufficiently seeded and avoids stalls in entropy-dependent operations.

On modern hardware with reliable RNG support (such as `RDRAND` or `RDSEED`), `haveged` may be less critical.   
It remains useful on older systems, virtual machines, or environments where entropy exhaustion has been observed.

**Install haveged**
```
apt-get install -y haveged
```
```
tee /etc/default/haveged >/dev/null <<'EOF'
DAEMON_ARGS="-w 1024"
EOF
```
```
systemctl daemon-reload
```
```
systemctl enable haveged && systemctl start haveged
```
**Verify:**
```
cat /proc/sys/kernel/random/entropy_avail
```
```
systemctl status haveged --no-pager
```

#### 3. PVE Processor Microcode

Processor microcode is a layer of low-level software that runs on the CPU and can be updated to apply vendor-provided fixes. Microcode updates can address hardware errata, improve stability, and mitigate certain security vulnerabilities at the processor level. In enterprise environments, keeping microcode current is an important part of maintaining a secure and reliable virtualization platform.

Microcode can often be updated through the operating system, but availability and update mechanisms vary by platform and CPU vendor. Some systems also rely on firmware update mechanisms such as Intel Management Engine (ME) or AMD Platform Security Processor (PSP). Consult your server platform and CPU vendor documentation to confirm supported update paths and any required coordination with BIOS/firmware updates.

> **Performance**
>
> Microcode updates can resolve CPU errata and improve stability, and in some cases can improve performance by correcting inefficient behavior or improving instruction handling.
>
> **Trade-off**
>
> Microcode updates can change CPU behavior and may affect performance characteristics in certain workloads. Updates should be tested on a staging node first, applied during a maintenance window, and paired with a controlled reboot plan.

Execute within the Proxmox shell.

To run the script, use the following command **only** in the Proxmox VE Shell:

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/microcode.sh)"
```

> This script installs or updates the appropriate Intel or AMD microcode packages for the host.

Reboot the host after installation so the updated microcode can be applied.
> If VMs or containers are running, schedule a maintenance time to reboot.

After rebooting, verify that microcode updates are in effect:

```
journalctl -k | grep -E "microcode" | head -n 1
```
> This will give a result such as: `Jan 06 16:22:39 alphonsus kernel: microcode: Current revision: 0x00000028`

**Source:**  
[PVE Processor Microcode](https://community-scripts.github.io/ProxmoxVE/scripts?id=microcode&category=Proxmox+%26+Virtualization)

---

## III. Virtual Machine CPU and Memory Configuration

This section outlines **optional, environment-dependent considerations** for configuring virtual machine CPU and memory settings in Proxmox VE. The guidance provided here is not intended to define defaults or broadly applicable best practices. Instead, it highlights configuration choices that may be beneficial for certain workloads, performance profiles, or operational models.

CPU and memory behavior in virtualized environments is highly sensitive to workload characteristics, hardware capabilities, and cluster design. As a result, the topics in this section should be evaluated individually and applied only when there is a clear performance, compatibility, or operational justification (such as live migration requirements or cluster scalability goals).

---

#### 1. Hugepages

Hugepages reduce memory management overhead by using larger page sizes (typically 2MB pages instead of 4KB pages). They can improve performance for workloads that:

- allocate large contiguous memory regions
- have high TLB pressure
- are highly memory intensive

Potential benefits:

- lower CPU overhead for memory translation
- improved performance for certain databases and high-throughput workloads
- more predictable memory behavior in some cases

Trade-offs:

- memory becomes less flexible and harder to reclaim
- hugepages must be planned and reserved
- over-allocation can reduce available memory for other workloads

> Hugepages should be treated as an advanced optimization. Only enable them when there is a measured performance need and when you understand the memory reservation and operational impacts.

---

**Why HugePages require planning and reservation**

HugePages must be planned and reserved (usually at boot time) because they provide large, contiguous memory blocks for performance-critical workloads. By using larger page sizes (such as 2MB or 1GB), HugePages reduce CPU overhead by lowering Translation Lookaside Buffer (TLB) misses and improving virtual-to-physical address translation efficiency. They are commonly used by large databases, high-throughput networking stacks (such as DPDK), and other memory-intensive applications that benefit from predictable, low-latency memory behavior.

Unlike normal memory pages, HugePages are not dynamically freed and refilled easily. HugePages are pinned (non-swappable) and must be reserved in advance so the kernel can allocate and track the required number of large pages. In many cases, adjusting HugePages allocations requires kernel parameter changes and may require a reboot to take effect reliably, especially when using 1GB pages or when memory fragmentation prevents allocation at runtime.

**Why planning and reservation are necessary**

- **Performance:** Reduces TLB misses by using larger pages (for example 2MB or 1GB), improving address translation speed and reducing CPU overhead in memory-intensive workloads.
- **Contiguity:** Provides large, uninterrupted memory blocks, which can be important for Direct Memory Access (DMA), packet processing (DPDK), large caches, and database buffer pools.
- **Non-swappable:** HugePages are pinned and cannot be swapped to disk, avoiding severe performance hits that occur when large memory regions are swapped under pressure.
- **Kernel limits:** The kernel needs to know how many HugePages to reserve, and in many cases they cannot be added or removed easily after boot without rebooting due to fragmentation or allocation constraints.

**How HugePages are typically configured**

- **Calculate needs:** Determine how much memory the workload should consume as HugePages (for example, Oracle SGA size / HugePage size).
- **Reserve hugepages using kernel parameters:** Configure the number of HugePages to allocate using parameters such as `vm.nr_hugepages` (2MB pages) and, when applicable, page size parameters such as `hugepagesz`.
  - Example configuration location: `/etc/sysctl.conf` or `/etc/sysctl.d/99-hugepages.conf`
- **Optional bootloader configuration:** For large page sizes (such as 1GB) or for deterministic reservation behavior, set HugePages via GRUB kernel command line parameters such as:
  - `default_hugepagesz=1G hugepagesz=1G hugepages=4`
- **Mount `hugetlbfs` (if required):** Some applications require access to HugePages through a mounted filesystem interface.
- **Apply changes:** Use `sysctl -p` for sysctl-managed settings, or reboot the host for kernel command line reservations and to avoid allocation failures due to fragmentation.
- **Disable Transparent HugePages (THP) if necessary:** Many database vendors recommend disabling THP (for example, `transparent_hugepage=never` in GRUB), because THP can introduce fragmentation and unpredictable latency spikes under memory pressure.

**Key takeaway**

HugePages must be planned based on peak workload requirements and reserved in advance. If insufficient HugePages are reserved, applications may fail to start, fail to allocate large contiguous buffers, or lose the performance benefits that HugePages are intended to provide.


---

**Host-level hugepages (Proxmox VE host configuration)**

Hugepages must be made available at the host kernel level before VMs can reliably consume them. This is typically done by configuring kernel parameters via GRUB so the host reserves hugepage memory during boot.

A common configuration is to enable 1GB hugepages for large VMs while leaving the default hugepage size at 2MB for general compatibility:

Edit GRUB:

```
nano /etc/default/grub
```
> Don't forget to prepend `sudo` if using a non-root user.

Example configuration:

`GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on hugepagesz=1G default_hugepagesz=2M"`

This example enables 1GB hugepages while keeping 2MB hugepages as the default size for compatibility, and it also enables Intel IOMMU support for passthrough and advanced device isolation use cases.

- `intel_iommu=on` enables the Intel IOMMU (Input–Output Memory Management Unit). This is commonly required for PCI passthrough (`VFIO`), SR-IOV, and strong DMA isolation. If your environment does not use passthrough or SR-IOV, this option is not strictly required.
- `hugepagesz=1G` enables 1GB hugepages on the host.
- `default_hugepagesz=2M` ensures the system still uses 2MB hugepages as the default hugepage size when hugepages are requested without a specific size, preserving compatibility.

> Proxmox VE typically defaults to `GRUB_CMDLINE_LINUX_DEFAULT="quiet"` to keep the kernel command line minimal. Additional parameters should be added only when required by your environment.


Update GRUB and reboot:

```
update-grub
```
> Don't forget to prepend `sudo` if using a non-root user.  
> If using a non-root user without `sudo` then you may get a response saying the command does not exist.
```
reboot now
```

Verify hugepage availability after reboot:

```
cat /proc/meminfo | grep -i Huge
```

> Hugepages are reserved at boot and are not swappable. The reserved memory becomes unavailable to normal host processes unless it is used by hugepage-backed workloads. Plan capacity carefully and avoid reserving excessive hugepages.

---

**VM-specific hugepages (per-VM configuration)**

Hugepages are typically enabled on a per-VM basis so only selected workloads consume reserved hugepage-backed memory.

To configure hugepages for a VM:

1. Stop the VM (required for hugepage changes)

2. Enable the 1GB hugepage CPU flag (required for 1GB hugepages)  

In the Proxmox Web UI:
- VM → Hardware → CPU → enable `+pdpe1gb`  
> Make sure to toggle Advanced settings as enabled (check box).  
> The `pdpe1gb` CPU flag enables the guest to use 1GB pages. Without it, 1GB hugepages will not be usable inside the guest.

<img src="../images/enable_pdpe1gb_cpu_flag.png" alt="Enable pdpe1gb CPU flag." height="482" width="695">

3. Add the `hugepages` setting to the VM configuration

HugePages can be enabled per-VM by adding a `hugepages` setting to the VM configuration. This ensures the VM’s memory is backed by HugePages from the host’s reserved pool.

This can be done by editing the VM configuration file or via the `qm set` command.

**Option A: Configure via VM config file**

VM configuration files are stored under:

`/etc/pve/qemu-server/<vmid>.conf`

Edit the VM config file:

```
nano /etc/pve/qemu-server/<vmid>.conf
```
> Make sure to set the correct `vmid` for the `.conf` file.

Add one of the following examples:

- **2MB HugePages**
  - `hugepages: 2`
- **1GB HugePages**
  - `hugepages: 1`

If you want to keep the HugePages reserved after shutdown (optional), add:

- `keephugepages: 1`

Example configuration (2MB HugePages with reservation kept after shutdown):

- `hugepages: 2`
- `keephugepages: 1`

> When using `hugepages: 2`, Proxmox will attempt to allocate VM memory from 2MB hugepages.  
> When using `hugepages: 1`, Proxmox will attempt to allocate VM memory from 1GB hugepages (requires host and guest support, including the `+pdpe1gb` CPU flag).  
> `keephugepages: 1` prevents Proxmox from deleting reserved hugepages after VM shutdown, which can reduce allocation delays and lower the risk of allocation failure due to memory fragmentation.

---

**Option B: Configure via `qm set`**

For **2MB HugePages**:

```
qm set <vmid> --hugepages 2
```
> Make sure to set the correct `vmid`  

For **1GB HugePages**:

```
qm set <vmid> --hugepages 1024
```
```
qm set <vmid> --hugepages any
```
> If the `--hugepages` is set to `any` then 1 GiB hugepages will be used if possible, otherwise the size will fall back to 2 MiB.

If you want Proxmox to keep the HugePages reserved after shutdown (optional):

```
qm set <vmid> --keephugepages 1
```

Example (2MB HugePages with reservation kept after shutdown):

```
qm set <vmid> --hugepages 2 --keephugepages 1
```

> `--keephugepages 1` prevents Proxmox from deleting reserved hugepages after VM shutdown, which can reduce allocation delays and lower the risk of allocation failure due to memory fragmentation. This also reduces hugepage availability for other workloads while the VM is powered off, so capacity planning is required.

---

**Sizing considerations**

- When using **1GB HugePages**, VM memory should be configured as a multiple of 1GB to avoid allocation waste or start failures.
- If insufficient HugePages are available on the host, the VM may fail to start.
- HugePages are reserved and non-swappable, so plan capacity carefully.

> HugePages are not dynamically allocated under pressure. If the host does not have enough free HugePages available, the VM may fail to start or may block other HugePage-backed VMs from starting.


4. Start the VM and validate performance and stability

---

**Key considerations**

- **Performance:** Hugepages reduce CPU overhead for memory management and can improve performance for memory-intensive workloads such as databases, analytics platforms, and virtualization hosts running nested workloads.
- **Memory reservation:** Hugepages are allocated at boot and are not swappable. They reduce available RAM for the host and other VMs, so the reservation must be sized carefully.
- **VM start behavior:** If the host does not have enough available hugepages, the VM may fail to start. Hugepages should be treated as a reserved resource similar to pinned CPU or dedicated storage.
- **Compatibility:** 1GB hugepages require modern CPU support and the `+pdpe1gb` flag enabled on the VM. Mixed-hardware clusters must be validated carefully.
- **THP vs manual hugepages:** Manually configured hugepages are different from Transparent Hugepages (THP). THP can be disabled (`never`) on the host if it causes latency or fragmentation issues for specific workloads.

---

**Operational best practices**

- Use hugepages only when there is a measured benefit for the workload
- Prefer enabling hugepages for specific large VMs rather than broadly across the cluster
- Avoid over-reserving hugepages, as unused reserved memory reduces operational flexibility
- Validate VM start and failover behavior when hugepages are used
- Document hugepage allocations as part of capacity planning and maintenance procedures

---

#### Recommended baseline (enterprise default)

For most enterprise Proxmox clusters:

- Use a consistent cluster-compatible CPU model across nodes
- Avoid `host` passthrough unless required
- Enable NUMA only for large VMs where it provides measurable benefit
- Use fixed memory allocation for critical workloads
- Use ballooning selectively and only with monitoring
- Avoid aggressive memory overcommit in production
- Consider hugepages only for specific workloads with demonstrated benefit

---

## IV. Network Interface Optional Practices

This section outlines **optional, situational network-related adjustments** that may be useful in certain Proxmox VE environments. The practices described here are not default configurations and are not expected to be applicable to most deployments. They are primarily intended for troubleshooting, performance tuning, or mitigating specific, observed network issues.

Network behavior is highly dependent on physical infrastructure, kernel behavior, upstream connectivity, and workload traffic patterns. As a result, each item in this section should be evaluated independently and implemented only when there is a clear, documented need and an understanding of the potential trade-offs.

Topics include:

- Forcing APT to use IPv4 in environments where IPv6 connectivity causes slow or unreliable package operations
- Applying network-related `sysctl` adjustments to address specific performance or stability issues
- Enabling TCP BBR congestion control and TCP Fast Open for select traffic patterns
- Optimizing network interface queue length and making those settings persistent when addressing known throughput or latency constraints

---

#### 1. Force APT to use IPv4 (when IPv6 causes slow or flaky updates)
If your ISP, router, DNS configuration, or other network component makes IPv6 unreliable, it can cause issues or slow things down.  
**However, we can fix this by forcing IPv4 and it instantly makes updates consistent again.**
```
echo 'Acquire::ForceIPv4 "true";' | tee /etc/apt/apt.conf.d/99force-ipv4
```
**Verify it applied:**
```
cat /etc/apt/apt.conf.d/99force-ipv4
```
**To roll it back:**
```
rm -f /etc/apt/apt.conf.d/99force-ipv4
```
> Prepend `sudo` if using a non-root user

#### 2. Apply network sysctl optimizations

If you have a lot of network traffic (large storage networks, high east-west VM traffic, or generally want tighter and more predictable networking behavior), these sysctl tweaks can help improve throughput, reduce latency spikes, and harden common IPv4 behaviors.

Open/create the network performance tuning file `/etc/sysctl.d/99-network-performance.conf`:
```
nano /etc/sysctl.d/99-network-performance.conf
```
> Replace `nano` with `vi`, `vim`, or another editor if you prefer a different editor.

Paste the following contents into the file:
```
# Proxmox-safe network performance tuning

# Core network buffers
net.core.netdev_max_backlog = 8192
net.core.optmem_max = 8192
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.somaxconn = 8151

# IPv4 hardening
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.log_martians = 0
net.ipv4.conf.all.rp_filter = 1

net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.log_martians = 0
net.ipv4.conf.default.rp_filter = 1

# ICMP sanity
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Ephemeral port range
net.ipv4.ip_local_port_range = 1024 65535

# TCP behavior tuning
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 240
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_limit_output_bytes = 65536
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_rfc1337 = 1
net.ipv4.tcp_sack = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_synack_retries = 2

# TCP memory buffers
net.ipv4.tcp_rmem = 8192 87380 16777216
net.ipv4.tcp_wmem = 8192 65536 16777216

# UNIX socket queue depth
net.unix.max_dgram_qlen = 4096
```

> **Core network buffer and queue tuning**
>
> `net.core.netdev_max_backlog = 8192`  
> What it is: Maximum number of packets queued on the kernel receive queue when the NIC receives traffic faster than it can be processed.  
> Default / range: Default is commonly `1000` on many distros; practical range depends on memory and traffic patterns.  
> Performance: Reduces packet drops during bursts (VM traffic spikes, backups, replication, Ceph) and improves throughput stability under load.  
> Trade-off: Slightly higher memory usage during sustained backlogs; if set excessively high, it can hide CPU saturation while increasing queueing latency.
>
> `net.core.optmem_max = 8192`  
> What it is: Maximum optional memory allowed per socket (used for certain socket options and ancillary data).  
> Default / range: Defaults vary by kernel; generally small by default and not frequently tuned.  
> Performance: Provides modest per-socket headroom and can help avoid overly tight socket option memory limits under some workloads.  
> Trade-off: Minimal risk; slightly increases per-socket memory allowance with negligible system impact.
>
> `net.core.rmem_max = 16777216` and `net.core.wmem_max = 16777216`  
> What it is: System-wide maximum receive and send buffer sizes per socket (bytes).  
> Default / range: Defaults are often much lower (commonly in the low MB range or below); must be >= the max values used by TCP autotuning.  
> Performance: Allows TCP autotuning and applications to scale buffers for high throughput or higher latency paths (replication, backups, large transfers).  
> Trade-off: Increased potential memory consumption if many sockets scale up simultaneously.
>
> `net.core.somaxconn = 8151`  
> What it is: Maximum pending connection backlog for listen sockets (server accept queue).  
> Default / range: Default is commonly `4096` (sometimes `128` on older systems); effective maximum may be capped by the application’s listen backlog.  
> Performance: Helps services survive connection bursts without refusing clients (API endpoints, web services, reverse proxies).  
> Trade-off: Low risk, but only helps if the application also requests a larger backlog.
>
> **IPv4 hardening and routing correctness**
>
> `net.ipv4.conf.all.accept_redirects = 0` and `net.ipv4.conf.default.accept_redirects = 0`  
> What it is: Controls whether ICMP redirects are accepted (messages claiming a better gateway exists).  
> Default / range: Boolean (`0` or `1`); defaults vary by distro and interface type.  
> Performance: Improves routing determinism and reduces the chance of routing instability caused by redirects.  
> Trade-off: May impact environments that rely on redirects (uncommon and generally discouraged).
>
> `net.ipv4.conf.all.secure_redirects = 0` and `net.ipv4.conf.default.secure_redirects = 0`  
> What it is: Controls acceptance of “secure” redirects from known gateways.  
> Default / range: Boolean (`0` or `1`); defaults vary.  
> Performance: Reduces unexpected route changes and improves deterministic behavior.  
> Trade-off: May affect legacy networks that depend on redirects.
>
> `net.ipv4.conf.all.send_redirects = 0` and `net.ipv4.conf.default.send_redirects = 0`  
> What it is: Controls whether this host sends ICMP redirects to other systems.  
> Default / range: Boolean (`0` or `1`); commonly enabled on systems that forward traffic.  
> Performance: Prevents accidental routing advisory behavior and reduces misrouting risk.  
> Trade-off: No practical downside for typical Proxmox deployments that are not acting as routers.
>
> `net.ipv4.conf.all.accept_source_route = 0` and `net.ipv4.conf.default.accept_source_route = 0`  
> What it is: Controls acceptance of source-routed packets where the sender specifies the route.  
> Default / range: Boolean (`0` or `1`); typically already `0` on modern distros.  
> Performance: Not performance-driven; this is a correctness and security hardening control.  
> Trade-off: No functional downside in modern environments.
>
> `net.ipv4.conf.all.log_martians = 0` and `net.ipv4.conf.default.log_martians = 0`  
> What it is: Controls logging of “martian” packets (invalid or suspicious source/destination addresses).  
> Default / range: Boolean (`0` or `1`); often `0` by default on many distros.  
> Performance: Reduces log noise and prevents disk growth from excessive kernel logging in busy environments.  
> Trade-off: Reduced forensic visibility when diagnosing spoofing or routing issues.
>
> `net.ipv4.conf.all.rp_filter = 1` and `net.ipv4.conf.default.rp_filter = 1`  
> What it is: Reverse path filtering validates that the source IP is reachable via the interface it arrived on.  
> Default / range: Typically `0`, `1`, or `2` depending on distro; `1` is strict mode, `2` is loose mode.  
> Performance: Improves routing correctness and provides strong spoofing protection in typical single-path routing setups.  
> Trade-off: Can break asymmetric routing, advanced multi-homing, VRFs, and policy routing; consider `2` (loose) if you have complex routing requirements.
>
> **ICMP sanity controls**
>
> `net.ipv4.icmp_echo_ignore_broadcasts = 1`  
> What it is: Ignores ICMP echo requests sent to broadcast addresses.  
> Default / range: Boolean (`0` or `1`); often `1` by default on modern systems.  
> Performance: Reduces noise and mitigates classic amplification patterns.  
> Trade-off: No practical downside.
>
> `net.ipv4.icmp_ignore_bogus_error_responses = 1`  
> What it is: Ignores malformed or noncompliant ICMP error responses.  
> Default / range: Boolean (`0` or `1`); often `1` by default on modern systems.  
> Performance: Improves robustness and reduces unnecessary error handling.  
> Trade-off: Minimal risk; may ignore some malformed diagnostics.
>
> **Ephemeral port range expansion**
>
> `net.ipv4.ip_local_port_range = 1024 65535`  
> What it is: Defines the range of ports available for outbound (ephemeral) connections.  
> Default / range: Usually something like `32768 60999` or similar by default; range must be within `1024–65535`.  
> Performance: Reduces the risk of ephemeral port exhaustion in high-connection workloads (reverse proxies, NAT, many short-lived outbound connections).  
> Trade-off: Slightly increases the number of ports in use over time; generally a standard server tuning.
>
> **TCP behavior tuning (latency and connection resilience)**
>
> `net.ipv4.tcp_fin_timeout = 10`  
> What it is: Time a socket remains in FIN-WAIT-2 after close.  
> Default / range: Default is commonly `60` seconds; valid values are positive integers (seconds).  
> Performance: Frees resources faster on high-churn servers and reduces stale connection buildup.  
> Trade-off: Less tolerant of slow or misbehaving peers; could drop lingering connections sooner.
>
> `net.ipv4.tcp_keepalive_time = 240`, `net.ipv4.tcp_keepalive_intvl = 30`, `net.ipv4.tcp_keepalive_probes = 3`  
> What it is: How idle TCP connections are probed to detect dead peers.  
> Default / range: Defaults are commonly `7200` (time), `75` (interval), and `9` (probes); values are integers in seconds/count.  
> Performance: Cleans up dead connections faster and recovers more quickly after network interruptions.  
> Trade-off: Increased keepalive traffic and potentially more frequent disconnect detection on unstable links.
>
> `net.ipv4.tcp_max_syn_backlog = 8192`  
> What it is: Queue size for half-open TCP connections during the handshake.  
> Default / range: Default is often `2048` or `4096`; valid values are integers.  
> Performance: Improves resilience to connection spikes and SYN flood scenarios.  
> Trade-off: Minor memory increase for queued handshakes.
>
> `net.ipv4.tcp_limit_output_bytes = 65536`  
> What it is: Caps per-socket queued output before TCP throttles.  
> Default / range: Default varies by kernel; integer bytes.  
> Performance: Improves fairness and reduces latency caused by excessive buffering per flow (can help mitigate bufferbloat).  
> Trade-off: May slightly reduce peak throughput for single large flows on very fast links.
>
> `net.ipv4.tcp_mtu_probing = 1`  
> What it is: Enables adaptive MTU probing when Path MTU Discovery is unreliable.  
> Default / range: Typically `0` (off) by default; values are `0`, `1`, or `2` depending on kernel behavior.  
> Performance: Improves reliability and reduces black-hole stalls across tunnels, VPNs, and misconfigured networks.  
> Trade-off: Adds minor probing overhead; usually negligible.
>
> `net.ipv4.tcp_rfc1337 = 1`  
> What it is: Protects against TIME-WAIT assassination edge-cases.  
> Default / range: Boolean (`0` or `1`); often `0` by default.  
> Performance: Improves long-lived connection correctness and stability.  
> Trade-off: No practical downside.
>
> `net.ipv4.tcp_sack = 1`  
> What it is: Enables Selective ACKs for efficient retransmissions.  
> Default / range: Boolean (`0` or `1`); usually already `1` by default on modern kernels.  
> Performance: Improves throughput and recovery on lossy or congested links.  
> Trade-off: Minimal downside; should generally remain enabled.
>
> `net.ipv4.tcp_slow_start_after_idle = 0`  
> What it is: Controls whether TCP resets congestion window after idle periods.  
> Default / range: Boolean (`0` or `1`); commonly `1` by default.  
> Performance: Maintains throughput for long-lived connections that pause briefly (replication, backups, clustered services).  
> Trade-off: Can increase congestion risk on oversubscribed or fragile networks.
>
> `net.ipv4.tcp_syn_retries = 3` and `net.ipv4.tcp_synack_retries = 2`  
> What it is: How many retries occur before giving up on connection setup.  
> Default / range: Defaults are commonly `6` and `5`; values are integers.  
> Performance: Faster failure detection when endpoints are unreachable.  
> Trade-off: Less tolerant of very high-latency or packet-loss links; may fail connections sooner in degraded networks.
>
> **TCP memory buffer ranges (autotuning limits)**
>
> `net.ipv4.tcp_rmem = 8192 87380 16777216` and `net.ipv4.tcp_wmem = 8192 65536 16777216`  
> What it is: Minimum, default, and maximum TCP buffer sizes used by autotuning (bytes).  
> Default / range: Defaults vary by kernel; the three values are `min default max`, and max must not exceed `rmem_max/wmem_max`.  
> Performance: Higher maximums improve throughput on high bandwidth-delay product paths (WAN replication, backup streams, routed storage networks).  
> Trade-off: Increased memory usage if many connections scale toward the maximum; monitor socket memory usage on high-connection-count hosts.
>
> **UNIX socket queue depth**
>
> `net.unix.max_dgram_qlen = 4096`  
> What it is: Maximum queued datagrams for UNIX domain sockets.  
> Default / range: Default is commonly `10`; integer values.  
> Performance: Reduces message loss under bursty local IPC workloads (logging, agents, container tooling, daemon communication).  
> Trade-off: More memory can be consumed during local message backlogs if consumers fall behind.


Save and exit the editor.
> - **nano:** Press `Ctrl + X`, then press `Y` to confirm saving changes, then press `Enter` to confirm the filename.
> - **vi / vim:** Press `Esc`, type `:wq`, then press `Enter` to write and quit.

**Create a backup of the current sysctl network configurations prior to applying the new sysctl settings**
Create the backup script `network_sysctl_snapshot.sh`:
```
nano network_sysctl_snapshot.sh
```
> Replace `nano` with `vi`, `vim`, or another editor if you prefer a different editor.  
> We put this in a scripts directory inside of home. i.e. `nano $HOME/scripts/network_sysctl_snapshot.sh`
```
#! /bin/bash
set -euo pipefail

TWEAK_FILE="/etc/sysctl.d/99-network-performance.conf"
BACKUP_DIRECTORY_PATH="$HOME/backups"
BACKUP_FILENAME="network-sysctl-backup-$(date +%F_%H%M%S).conf"
BACKUP_FILEPATH="$BACKUP_DIRECTORY_PATH/$BACKUP_FILENAME"

# Create backups directory if it doesn't exist
mkdir -p $BACKUP_DIRECTORY_PATH

# Ensure the tweak file exists
if [[ ! -f "$TWEAK_FILE" ]]; then
  echo "ERROR: Tweak file not found: $TWEAK_FILE" >&2
  exit 1
fi

echo "# Backup of current sysctl values BEFORE applying $TWEAK_FILE" > "$BACKUP_FILEPATH"
echo "# Created: $(date)" >> "$BACKUP_FILEPATH"
echo "# Host: $(hostname -f 2>/dev/null || hostname)" >> "$BACKUP_FILEPATH"
echo >> "$BACKUP_FILEPATH"

# Extract only valid sysctl keys from the tweak file and back up their current running values
grep -vE '^\s*#|^\s*$' "$TWEAK_FILE" \
  | awk -F= '{gsub(/[[:space:]]+/, "", $1); print $1}' \
  | while read -r key; do
      # Run sysctl for the current live value and write output as "key = value"
      value="$(sysctl -n "$key" 2>/dev/null || true)"

      if [[ -z "$value" ]]; then
        echo "# WARNING: Could not read current value for: $key (may not exist on this kernel)" >> "$BACKUP_FILEPATH"
      else
        echo "$key = $value" >> "$BACKUP_FILEPATH"
      fi
    done

echo "Backup complete: $BACKUP_FILEPATH"
```
Save and exit the editor.
> - **nano:** Press `Ctrl + X`, then press `Y` to confirm saving changes, then press `Enter` to confirm the filename.
> - **vi / vim:** Press `Esc`, type `:wq`, then press `Enter` to write and quit.

Execute script:
```
bash scripts/network_sysctl_snapshot.sh
```
> Make sure to use the filepath you set  
> Optional: Make executable `chmod +x scripts/network_sysctl_snapshot.sh`  
> Optional: Run `./scripts/network_sysctl_snapshot.sh`  
>
> You will see an output like: `Backup complete: /root/backups/network-sysctl-backup-2025-12-23_174434.conf`

Apply the new sysctl settings immediately:
```
sysctl --system
```

> This reloads all sysctl configuration files and applies the new values to the running kernel without requiring a reboot.

#### 3. Enable TCP BBR and Fast Open

This tweak can improve throughput and reduce latency in certain conditions, especially over longer RTT links and congested network paths. BBR is a modern congestion control algorithm that can outperform traditional algorithms in high-latency or lossy networks, while TCP Fast Open can reduce handshake overhead for repeat connections.

> **Performance**
>
> BBR can improve throughput and reduce buffering delays on links where traditional congestion control algorithms struggle, such as WAN links, VPN tunnels, and congested paths. TCP Fast Open can reduce connection setup latency for services with frequent reconnects by allowing data to be sent during the initial handshake.
>
> **Trade-off**
>
> BBR may not be ideal on all networks and can cause unfair bandwidth sharing in some mixed congestion-control environments. TCP Fast Open is not universally supported across middleboxes and may be blocked or ignored by some firewalls, load balancers, or NAT devices.

**Create the BBR sysctl configuration:**

```
echo "net.core.default_qdisc = fq" | tee -a /etc/sysctl.d/99-tcp-bbr.conf
```

```
echo "net.ipv4.tcp_congestion_control = bbr" | tee -a /etc/sysctl.d/99-tcp-bbr.conf
```

**Enable TCP Fast Open:**

```
echo "net.ipv4.tcp_fastopen = 3" | tee -a /etc/sysctl.d/99-tcp-fastopen.conf
```

**Load the BBR kernel module:**

```
modprobe tcp_bbr
```

> If the module fails to load, your kernel may not include BBR support, or the module may already be built-in and not appear as a separate loadable module.

**Apply the sysctl settings:**

```
sysctl -p /etc/sysctl.d/99-tcp-bbr.conf
```

```
sysctl -p /etc/sysctl.d/99-tcp-fastopen.conf
```

**Verify it applied:**

```
sysctl net.ipv4.tcp_congestion_control
```

```
lsmod | grep bbr || true
```

#### 4. Optimize NIC queue length and make it persistent

This is a practical tuning tweak when you see bursty traffic, intermittent drops, or you want to increase the queue depth for a busy interface. Increasing the transmit queue length can help absorb short bursts of outbound traffic before packets are dropped, especially on virtualization hosts carrying many VM flows.

> **Performance**
>
> A higher transmit queue length can improve burst tolerance and reduce packet drops during short spikes in traffic. This is especially useful for busy bridged interfaces carrying VM and storage traffic, where many flows can briefly exceed what the host can immediately transmit.
>
> **Trade-off**
>
> Longer queues can increase latency under sustained congestion (bufferbloat), since packets may spend longer waiting before they are transmitted. This should be treated as a burst-handling improvement, not a substitute for addressing persistent congestion or link saturation.

**Find your interface:**

```
ip link show
```

**Set the transmit queue length (replace `ens192` with your interface name):**

```
ip link set ens192 txqueuelen 10000
```

**Persist the queue length across reboots using a udev rule:**

```
echo 'ACTION=="add", SUBSYSTEM=="net", KERNEL=="ens192", RUN+="/sbin/ip link set ens192 txqueuelen 10000"' | tee /etc/udev/rules.d/60-net-txqueue.rules
```

> This ensures the queue length is re-applied automatically whenever the interface is brought up, including after reboots.

**Ensure TCP timestamps are enabled:**

```
echo 'net.ipv4.tcp_timestamps = 1' | tee -a /etc/sysctl.d/99-network-performance.conf
```

```
sysctl -p /etc/sysctl.d/99-network-performance.conf
```

> TCP timestamps help improve RTT measurement and can improve loss recovery and performance in many environments. They are typically enabled by default on modern systems, but this ensures consistency.

**Verify it applied (replace `ens192` with your interface name):**

```
ip -s link show ens192 | head -n 20
```

```
cat /etc/udev/rules.d/60-net-txqueue.rules
```

---

## V. Storage and Disk Configuration Optional Practices

This section outlines **optional, environment-dependent storage and disk configuration considerations** for Proxmox VE. The items covered here are not intended to represent default configurations or universally applicable best practices. Instead, they highlight storage-related options and tuning decisions that may be beneficial in specific workloads, performance-sensitive environments, or non-standard storage architectures.

Storage behavior in virtualized environments varies significantly based on underlying hardware, storage backends, workload I/O patterns, and resiliency requirements. As such, each topic in this section should be evaluated individually and implemented only when there is a clear performance, operational, or architectural justification.

---

#### 1. Log2RAM (for consumer SSDs)

Many enterprise environments still deploy Proxmox on consumer-grade SSDs in labs, edge locations, or cost-sensitive tiers, and these drives typically have lower write endurance than enterprise storage. Log writes can be frequent and continuous, especially on virtualization hosts running many services and guests. Log2RAM reduces SSD wear by writing `/var/log` to a RAM-backed filesystem (tmpfs) and syncing logs back to disk periodically.

> **Performance**
>
> Writing logs to RAM reduces disk I/O and can improve responsiveness on systems where the root disk is a bottleneck. It also significantly reduces write amplification on consumer SSDs by keeping frequent small writes off the disk.
>
> **Trade-off**
>
> Logs stored in RAM are not guaranteed to survive a sudden power loss, kernel panic, or crash between sync intervals. This reduces forensic and troubleshooting visibility for the period since the last sync. Log2RAM also consumes memory, so it should be sized appropriately for the host and tested before enabling broadly.

The script below installs Log2RAM and configures it for Proxmox. 
> It is recommended you test on a non-production node before applying to production if possible.
```
#!/usr/bin/env bash
# Install and configure Log2RAM on Proxmox
set -euo pipefail

LOG2RAM_SIZE="${LOG2RAM_SIZE:-256M}"
JOURNAL_MAX_USE="${JOURNAL_MAX_USE:-50M}"

if [[ "${EUID:-$(id -u)}" -ne 0 ]]; then
  echo "ERROR: Please run as root (use sudo)." >&2
  exit 1
fi

# Validate size formats
if ! [[ "$LOG2RAM_SIZE" =~ ^[0-9]+[KMG]$ ]] || ! [[ "$JOURNAL_MAX_USE" =~ ^[0-9]+[KMG]$ ]]; then
  echo "ERROR: Invalid size format. Use format like 256M, 512M, 1G" >&2
  exit 1
fi

TMPDIR="$(mktemp -d)"
trap "rm -rf $TMPDIR" EXIT

# Install dependencies
echo "[log2ram] Installing dependencies..."
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y rsync curl ca-certificates

# Check if already installed
if command -v log2ram >/dev/null 2>&1 || systemctl list-unit-files | grep -q "^log2ram\.service"; then
  echo "[log2ram] Already installed, skipping download"
else
  echo "[log2ram] Downloading and installing..."
  curl -fsSL -o "$TMPDIR/log2ram.tar.gz" \
    "https://github.com/azlux/log2ram/archive/refs/heads/master.tar.gz"
  tar -xzf "$TMPDIR/log2ram.tar.gz" -C "$TMPDIR"
  cd "$TMPDIR"/log2ram-* && bash ./install.sh </dev/null
fi

# Configure log2ram
if [[ -f /etc/log2ram.conf ]]; then
  cp /etc/log2ram.conf "/etc/log2ram.conf.bak.$(date +%F_%H%M%S)"
  sed -i -E "s|^[#[:space:]]*SIZE=.*|SIZE=${LOG2RAM_SIZE}|" /etc/log2ram.conf
  sed -i -E "s|^[#[:space:]]*USE_RSYNC=.*|USE_RSYNC=true|" /etc/log2ram.conf
  echo "[log2ram] Configured: SIZE=${LOG2RAM_SIZE}, USE_RSYNC=true"
fi

# Configure journald limits
cp /etc/systemd/journald.conf "/etc/systemd/journald.conf.bak.$(date +%F_%H%M%S)"
sed -i -E "s|^[#[:space:]]*SystemMaxUse=.*|SystemMaxUse=${JOURNAL_MAX_USE}|" /etc/systemd/journald.conf
sed -i -E "s|^[#[:space:]]*RuntimeMaxUse=.*|RuntimeMaxUse=${JOURNAL_MAX_USE}|" /etc/systemd/journald.conf

# Enable and start
systemctl daemon-reload
systemctl enable log2ram
systemctl start log2ram
systemctl restart systemd-journald

echo "Log2RAM installed and configured"
echo "SIZE=${LOG2RAM_SIZE}, JOURNAL_MAX_USE=${JOURNAL_MAX_USE}"
echo "Reboot to fully activate tmpfs mount for /var/log"
```
> After installation, rebooting is recommended to ensure the tmpfs mount for `/var/log` is active early in the boot process. If you use centralized logging or a SIEM, ensure logs are forwarded off-host so log data is not lost during unexpected outages.

**Verify it applied:**
```
df -h | grep log2ram
```
```
mount | grep log2ram
```
```
systemctl status log2ram
```

---

### VI. Performance Versus Safety Trade-Offs

This section explains the trade-offs between performance optimizations and operational safety.

Key points include:
- Why Proxmox defaults are conservative
- When deviating from defaults may be justified
- Risks associated with aggressive performance tuning
- The importance of understanding storage and hardware guarantees before changing defaults

**Performance vs Safety Context**  
Proxmox and the underlying Linux kernel ship with conservative defaults by design. These defaults are intended to be safe across a wide range of hardware, storage backends, and workload types, including mixed VM and container environments. They favor predictability and survivability under unexpected conditions over maximum performance.
In some environments, especially where hardware characteristics and workloads are well understood, selectively deviating from these defaults can improve responsiveness and reduce performance issues. However, doing so almost always reduces safety margins and increases the risk of instability if assumptions about memory, storage, or workload behavior are incorrect.

#### 1. Optimize your memory management 

Even on larger systems, memory pressure events can cause noticeable latency or temporary unresponsiveness. On lower-RAM nodes, memory tuning can help keep the system responsive even when memory is constrained.

The following settings intentionally adjust how aggressively the kernel uses swap, buffers dirty pages, and allocates memory. These changes can smooth performance and reduce latency, but they also reduce the kernel’s ability to absorb extreme or unexpected memory pressure.

**What memory pressure events are**
Memory pressure events occur when the system is forced to aggressively reclaim memory to satisfy new allocations. This can involve increased page reclaim, writeback of dirty pages, swapping, memory compaction, or invoking the out-of-memory killer. On virtualization hosts, these events can manifest as latency spikes, temporary unresponsiveness, or stalled workloads, even when the system has not fully exhausted available RAM.

**Open/Create a new configuration file at `/etc/sysctl.d/`:**
```
nano /etc/sysctl.d/99-memory.conf
```
> replace nano with vi, vim, etc. if you prefer a different editor.

**Paste the following inside `/etc/sysctl.d/99-memory.conf`:**
```
# Balanced Memory Optimization
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.overcommit_memory = 1
vm.max_map_count = 262144
```
> **Lower swappiness**
>
> Swappiness controls how aggressively the kernel moves memory to swap. The default value on most Linux systems is 60, with a valid range from 0 to 100. Lower values tell the kernel to prefer reclaiming unused memory and caches before swapping application memory, reducing swap activity and avoiding the performance penalties of disk-backed memory. The trade-off is reduced flexibility under sustained memory pressure, which can increase the likelihood of abrupt out-of-memory conditions if memory demand remains high.
>
> **Dirty ratio tuning**
>
> Dirty ratio settings control how much modified (dirty) data the kernel allows to accumulate in memory before forcing it to be written to disk. The default values are typically `vm.dirty_ratio = 20` and `vm.dirty_background_ratio = 10`, expressed as percentages of system memory. Lowering these values results in earlier and more consistent writeback, avoiding large, sudden flushes that can cause latency spikes. The trade-off is that these settings assume storage can sustain steady writeback; slow or unreliable storage may still experience stalls under sustained write-heavy workloads.
>
> **Memory overcommit**
>
> Memory overcommit controls how strictly the kernel accounts for memory allocations. The default behavior  
(`vm.overcommit_memory = 0`) uses heuristic-based accounting to balance safety and compatibility. Setting this value to `1` allows applications to allocate memory without upfront reservation, improving compatibility and reducing allocation failures. The trade-off is increased risk at runtime: if multiple workloads attempt to use their full allocations simultaneously, the system may be forced to reclaim memory aggressively or invoke the out-of-memory killer.
>
> **Max map count**
>
> Max map count limits the number of memory mappings a single process may create. Older Linux defaults commonly used `65530`, while many modern distributions and Proxmox builds now default to higher values such as `262144`.
>
> Lower values can be sufficient for simpler workloads but may cause hard-to-diagnose failures in modern, mapping-heavy applications such as databases, JVM-based services, monitoring tools, or container platforms.
>
> Using a higher value like `262144` significantly reduces the risk of these failures with negligible performance or memory overhead, making it a safer and more future-proof choice for Proxmox hosts running diverse or high-density workloads.

Save and exit the editor.
> - **nano:** Press `Ctrl + X`, then press `Y` to confirm saving changes, then press `Enter` to confirm the filename.
> - **vi / vim:** Press `Esc`, type `:wq`, then press `Enter` to write and quit.

**Enable memory compaction if supported by the system**  
```
if [ -f /proc/sys/vm/compaction_proactiveness ]; then
  echo "vm.compaction_proactiveness = 20" | tee -a /etc/sysctl.d/99-memory.conf
fi
```
> **Memory compaction trade-off**
>
> Memory compaction controls how aggressively the kernel rearranges memory to create larger contiguous free regions.  
On systems that support it, `vm.compaction_proactiveness` typically defaults to `0`, meaning compaction is mostly reactive and only occurs when required. The valid range is `0` to `100`, with higher values causing the kernel to compact memory more proactively in the background.
>
> Increasing this value (for example, to `20`) can reduce allocation latency for workloads that require large contiguous memory blocks, such as virtual machines or hugepages. The trade-off is additional background CPU activity, which may slightly reduce overall throughput on busy hosts or systems already under CPU pressure.

**Apply the configuration immediately**
```
sysctl -p /etc/sysctl.d/99-memory.conf
```
> `sysctl -p /etc/sysctl.d/99-memory.conf` reads the specified file and applies the sysctl values to the running kernel without requiring a reboot. This allows you to activate and validate the changes immediately, while still ensuring they persist across reboots.  

**Verify that the settings are applied:**
```
sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio vm.overcommit_memory vm.max_map_count
```
> These settings are best applied on systems with predictable workloads, adequate monitoring, and known-good storage performance. They should be avoided on hosts with highly unpredictable workloads, unreliable storage, or minimal observability, where conservative defaults provide a safer operating envelope.

**To roll back these changes, delete the custom configuration file:**
```
rm -rf /etc/sysctl.d/99-memory.conf
```
> Prepend `sudo` if using a non-root user
**Reload sysctl settings:**
```
sysctl --system
```

---

### VII. Miscellaneous Operational Optional Practices

This section captures additional Proxmox VE operational considerations that do not fit cleanly into other categories.

Topics may include:
- Increasing system limits
- Managing or removing the Proxmox “no subscription” notification
- UI and usability adjustments with operational impact
- Other environment-specific best practices identified over time

---

#### 1. Increase system limits (file watchers and open files)
This is a practical improvement for hosts running lots of containers, monitoring tools, or anything that keeps many files open.   
It can also help avoid weird edge-case failures.
```
echo "fs.inotify.max_user_watches = 1048576" | tee /etc/sysctl.d/99-maxwatches.conf
```
```
echo "* soft nofile 1048576" | tee /etc/security/limits.d/99-limits.conf
```
```
sysctl --system
```

**Verify it applied:**
```
sysctl fs.inotify.max_user_watches
```
Check `hard` resource limit
```
ulimit -Hn
```
Check `soft` resource limit:
```
ulimit -Sn
```
Check the maximum number of open file descriptors:
```
ulimit -n
```
> Note: On Proxmox/Debian systems, ulimit -n may still show 1024 even after logging out and back in.  
This is expected behavior.
>
> The hard limit is raised, but the soft limit is not automatically increased.  
This does not indicate a failure.
>
> You can confirm the limit is correctly enforced by running the following in the same shell:
>
> `ulimit -n 1048576`  
> `ulimit -n`
>
> If the value updates to 1048576, PAM limits are working, the hard limit is enforced, and the system is behaving correctly.

---

### Closing Guidance

This guide is intended to offer **optional, experience-informed guidance** that may be useful in certain Proxmox VE environments. The practices described are not defaults, requirements, or universal recommendations. Proxmox VE is intentionally flexible, and workloads often differ significantly in architecture, risk tolerance, and operational constraints.

Administrators should evaluate each practice on its own merits and adopt it only when there is a clear technical, operational, or security rationale. Deviations, alternatives, or non-adoption are valid outcomes and should be considered part of intentional system design rather than exceptions.

---

## Resources

[Entropy (computing)](https://en.wikipedia.org/wiki/Entropy_(computing))  
[haveged](https://linux.die.net/man/8/haveged)  
[PVE Processor Microcode](https://community-scripts.github.io/ProxmoxVE/scripts?id=microcode&category=Proxmox+%26+Virtualization)  
