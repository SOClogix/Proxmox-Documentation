# Installation Guide: Proxmox VE on Debian 12 (Bookworm)

This installation guide documents the SOClogix-approved process for installing **Proxmox Virtual Environment (VE)** on top of an existing **Debian 12 (Bookworm)** installation.

The Debian-based installation method is intended for **advanced or specialized use cases** where greater control over the base operating system is required. This includes environments with strict OS baselines, custom partitioning requirements, pre-existing automation, or compliance-driven constraints that necessitate a minimal Debian installation prior to deploying Proxmox components.

This guide emphasizes precision and operational awareness, as installing Proxmox VE on Debian requires careful handling of repositories, kernels, networking, and system services to ensure long-term stability and supportability.

The guide covers:
- Preparing a minimal Debian 12 installation  
- Adding and validating Proxmox repositories  
- Installing Proxmox VE components and kernel  
- System validation and alignment with SOClogix standards  
- Considerations and limitations specific to Debian-based installs  

This document assumes strong Linux and Debian administration experience and should only be used when the ISO-based installation method is not suitable. It is designed to integrate seamlessly with the same post-installation, hardening, and operational practices defined elsewhere in this repository.

---

This guide is in the works and still being developed as a standard replacement for the Proxmox VE ISO installation process. 