## Proxmox VE Best Practices Guide — Overview and Structure

### Purpose and Scope

This guide documents SOClogix-recommended best practices for operating Proxmox VE in production environments. It focuses on practical security hygiene, stable performance defaults, and common operational decisions. This is not a CIS benchmark or compliance audit and does not attempt to enumerate every possible hardening control. Instead, it provides opinionated guidance for the configurations and decisions most commonly encountered in real-world deployments.

Deviations from these recommendations are permitted when justified by workload requirements or architectural constraints, but such deviations should be intentional and documented.

#### Out of Scope

This guide does not cover the following topics:

- Full CIS benchmarks or compliance audits
- Regulatory or standards-based compliance mapping
- Storage-vendor-specific tuning and optimizations
- Guest operating system hardening
- Application-level performance tuning

---

### 1. Host-Level Security Best Practices

This section covers baseline security hygiene that should be implemented on all Proxmox VE hosts unless there is a documented exception.

Topics include:

- SSH hardening, including disabling password authentication and requiring public key authentication
- Restricting or disabling direct root login over SSH
- Creating a local, non-root administrative user for console and emergency access
- Using sudo for privilege escalation and retaining root for break-glass scenarios only
- Limiting management access to trusted networks
- Avoiding exposure of the Proxmox Web UI to untrusted or public networks
- Maintaining system updates and defining a reboot strategy for kernel updates
- Ensuring time synchronization and basic logging practices
- Installing and maintaining CPU microcode updates using the Proxmox VE Processor Microcode Helper Script to address hardware-level security vulnerabilities and stability issues

---

### 2. Proxmox Web UI Permissions and Roles Best Practices

This section focuses on best practices for managing authorization within the Proxmox Web GUI, independent of the underlying authentication source.

Topics include:

- Understanding Proxmox roles and privilege separation
- Assigning permissions at the appropriate scope (Datacenter, Cluster, Node, Resource)
- Using group-based permission assignments instead of individual users
- Avoiding overuse of the PVEAdmin role
- Creating custom roles when default roles are overly permissive
- Ensuring permissions align with operational responsibilities

Authentication backends (such as Active Directory) are assumed to be configured separately and are not covered in this section.

---

### 3. Virtual Machine CPU and Memory Configuration Best Practices

This section provides guidance for CPU and memory settings that apply to the majority of virtual machine workloads.

Topics include:

- Recommended CPU types for most workloads and when to avoid host passthrough
- Live migration considerations related to CPU selection
- When and why to enable NUMA
- Memory allocation strategies, including ballooning considerations
- Overcommit guidance and when to avoid aggressive memory overcommit
- High-level discussion of hugepages and when they may be appropriate

---

### 4. Network Interface Best Practices

This section outlines recommended virtual networking configurations for performance and compatibility.

Topics include:

- Using VirtIO network interfaces for most workloads
- Situations where alternative network models may be required
- Multiqueue configuration considerations
- VLAN awareness and tagging considerations
- MTU consistency across hosts and networks

---

### 5. Storage and Disk Configuration Best Practices

This section provides guidance on disk configuration choices that affect performance, stability, and recoverability.

Topics include:

- Disk controller selection, including VirtIO-SCSI versus VirtIO block
- When to use single or multiple SCSI controllers
- Asynchronous I/O modes, including native, io_uring, and threaded I/O
- Disk cache settings, including default recommendations and risk trade-offs
- SSD emulation behavior and when it is meaningful
- Discard and TRIM support, including when it should be enabled or avoided

---

### 6. Performance Versus Safety Trade-Offs

This section explains the trade-offs between performance optimizations and operational safety.

Key points include:

- Why Proxmox defaults are conservative
- When deviating from defaults may be justified
- Risks associated with aggressive performance tuning
- The importance of understanding storage and hardware guarantees before changing defaults

---

### 7. Miscellaneous Operational Best Practices

This section captures additional Proxmox VE operational considerations that do not fit cleanly into other categories and may expand over time.

Topics may include:

- Managing or removing the Proxmox “no subscription” notification in supported and supportable ways
- Post-install cleanup and validation tasks
- UI and usability adjustments with operational impact
- Other environment-specific best practices identified over time

---

### Closing Guidance

This guide is intended to provide clear, practical defaults that work well for most environments. Proxmox VE is flexible by design, and not all workloads have identical requirements. Administrators should treat these recommendations as a strong baseline and adjust only when there is a clear operational or technical justification.

---

## Resources

[Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/scripts)