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

#### Keep in Mind
- The sections of this guide are not necessarily done in order of "do this first" to "do this last".
- Review each section and determine which items you will implement first.
- Do not implement an action just because it is listed in this guide.  
First, ensure it meets organization needs, policy requirements, etc.

---

### I. Before You Start

#### 1. Conduct Backups/Snapshots

Before making any major changes, be sure to at least do the following:

- Take a backup of any config file before you edit it
- Apply changes on one node first if this is a cluster
- Reboot only when you have a maintenance window
- After any sysctl change, verify it actually loaded

Here is a nice quick “checkpoint” command list that you can use to capture a few important configuration settings.

```bash
# Snapshot current state you can refer back to
cp -a /etc/sysctl.d /etc/sysctl.d.bak.$(date +%F)
cp -a /etc/apt/apt.conf.d /etc/apt/apt.conf.d.bak.$(date +%F)
cp -a /etc/logrotate.conf /etc/logrotate.conf.bak.$(date +%F)

# See current sysctl overrides
sysctl --system | tail -n 50 >> "current_sysctl_overrides_backup-$(date +%F_%H%M%S).txt"
```
Individual code-blocks for copying:
```
cp -a /etc/sysctl.d /etc/sysctl.d.bak.$(date +%F)
```
```
cp -a /etc/apt/apt.conf.d /etc/apt/apt.conf.d.bak.$(date +%F)
```
```
cp -a /etc/logrotate.conf /etc/logrotate.conf.bak.$(date +%F)
```
```
sysctl --system | tail -n 50 >> "current_sysctl_overrides_backup-$(date +%F_%H%M%S).txt"
```
> Exclude `>> current_sysctl_overrides_backup.txt` if you just want to see the output of current sysctl overrides.

If you want to setup a script to run the commands to run, then copy the above code-block with all of the commands and paste them into a new script which can be created, made executable, and ran via the following commands:  
Open / Create Script using nano:
```
nano checkpoint_snapshot.sh
```
> replace nano with vi, vim, etc. if you prefer a different editor.

Save and exit the editor.
> - **nano:** Press `Ctrl + X`, then press `Y` to confirm saving changes, then press `Enter` to confirm the filename.
> - **vi / vim:** Press `Esc`, type `:wq`, then press `Enter` to write and quit.

Make script executable:
```
chmod +x checkpoint_snapshot.sh
```
Run Script:
```
./checkpoint_snapshot.sh
```
> Make sure to change the filepath for `current_sysctl_overrides_backup.txt` if you want it to be written to a specific directory other than the current working directory.

#### 2. PVE Post Install Helper Script

This helper script provides a guided, interactive way to complete common post-install setup tasks on a Proxmox VE host. It includes options for managing Proxmox repositories (disabling the Enterprise repo, enabling the No-Subscription repo, adding/correcting sources, optionally enabling the test repo), disabling the subscription nag, updating Proxmox VE packages, and rebooting the system.

> **Performance**
>
> Ensures the host is using the correct and intended package repositories, receives updates reliably, and reduces configuration drift. Running updates early also helps avoid stability issues caused by outdated packages or missing fixes.
>
> **Trade-off**
>
> This script modifies system configuration and may change repository behavior. Enabling the test repository is not recommended for production environments. System updates and reboots should be planned carefully to avoid impacting running workloads.

Execute within the Proxmox shell.

> It is recommended to answer `yes` (`y`) to most options presented during the process.  
> Do not enable the Test Repository  
> The ` Disable subscription nag?` option for this script no longer works. Say yes anyways or the script ends.  
> Recommended to leave High Availability Enabled  
> Recommended to not use this scripts Proxmox Update feature. Run updates manually after.  
> Do not approve a reboot if VMs or containers are running on the host.  

**Repository options explained**
> - **No-Subscription Repository:** Recommended for most non-enterprise Proxmox deployments. Provides stable updates without requiring a paid subscription, but should still be treated like production updates and applied during a maintenance window.
> - **Test Repository:** Intended for testing and early validation of upcoming packages. This repository can introduce breaking changes and should generally be avoided on production hosts unless you have a specific reason and a rollback plan.
> - **Ceph Repository:** Only enable this if you are running Ceph (or explicitly plan to). It provides Ceph-related packages that align with Proxmox-supported versions. Enabling it unnecessarily can introduce additional packages and update scope that most standalone hosts do not need.


To run the script, use the following command **only** in the Proxmox VE Shell:

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

**Source:**  
[PVE Post Install Script](https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install&category=Proxmox+%26+Virtualization)  

#### 3. Run PVE Updates / Upgrades

Keeping Proxmox updated is one of the most important operational best practices. Updates deliver security fixes, kernel improvements, and stability patches for both Proxmox and the underlying Debian base.

This section provides a repeatable update workflow using a simple script. It also includes an optional kernel pinning step as a cautionary safeguard to reduce the risk of boot issues after a kernel update.

> **Performance**
>
> Regular updates keep the host stable, improve hardware compatibility, and ensure Proxmox services benefit from upstream bug fixes and performance improvements.
>
> **Trade-off**
>
> Updates (especially kernel updates) can introduce regressions or require reboots. Kernel pinning can reduce reboot risk but must be revisited during planned maintenance to ensure the host does not stay pinned indefinitely.

---

##### Step 1: Collect the current kernel version

Before creating the script, capture the host’s currently running kernel version:

```
uname -r
```

You will use this value to populate the `proxmox-boot-tool kernel pin` line in the update script. This pins the currently running kernel so that if a newly installed kernel causes a boot issue, you retain a known-good kernel selection during reboot.

> Kernel pinning should be treated as a cautious operational safeguard.  
> Update the pinned kernel intentionally during a maintenance cycle where someone can be physically present to recover the host if needed (for example, selecting an older kernel from the boot menu).

---

##### Step 2: Create the update script

Create the script at `/bin/run_updates`:

```
nano /bin/run_updates
```

> Replace `nano` with `vi`, `vim`, or another editor if you prefer a different editor.

Paste the following contents into the file. Replace the pinned kernel version with the output of `uname -r` from Step 1:

```
#! /bin/bash
apt-get autoclean
apt update
apt upgrade -y
apt autoremove -y
proxmox-boot-tool kernel pin <YOUR_CURRENT_KERNEL_VERSION>
```
> See below `Alternate script (no kernel pinning)` section for explanation of apt commands

Example (do not copy this example version unless it matches your `uname -r` output):

`proxmox-boot-tool kernel pin 6.8.12-10-pve`

Save and exit the editor.

> Save and exit the editor:
>
> - **nano:** Press `Ctrl + X`, then press `Y` to confirm saving changes, then press `Enter` to confirm the filename.
> - **vi / vim:** Press `Esc`, type `:wq`, then press `Enter` to write and quit.

---

##### Step 3: Make the script executable

```
chmod +x /bin/run_updates
```

---

##### Step 4: Run updates

Run the script:

```
run_updates
```
> This performs a standard upgrade cycle (update, upgrade, autoremove) and pins the current kernel version to reduce reboot risk from unexpected kernel behavior.  
> Prepend `sudo` if using a non-root user
---

##### Operational guidance for kernel pinning

Pinning the current kernel helps ensure you always have a known-good kernel available after updates. However, it also means you should periodically update the pinned kernel intentionally as part of a maintenance window. Treat kernel changes as a planned operational event.

If you have out-of-band management (such as iDRAC, iLO, or similar remote console access), you may not need to pin the kernel, since you can remotely recover the host by selecting an older kernel at boot without being physically present.

---

##### Alternate script (no kernel pinning)

If you have iDRAC or another remote console solution that provides full out-of-band access, you can use this simplified version:

```
#! /bin/bash
apt-get autoclean
apt update
apt upgrade -y
apt autoremove -y
```
> **Update script command breakdown**
>
> - `apt-get autoclean` removes outdated package files from the local APT cache. This helps reclaim disk space by deleting package files that can no longer be downloaded (obsolete versions).
>
> - `apt update` refreshes the package index from configured repositories. This does not install updates, but ensures the system knows what versions are available.
>
> - `apt upgrade -y` installs available package updates without removing packages. The `-y` flag automatically answers “yes” to prompts, allowing the upgrade to run non-interactively.
>
> - `apt autoremove -y` removes packages that were automatically installed as dependencies but are no longer required. This helps keep the system clean and reduces disk usage over time.



---

### II. Host-Level Security Best Practices

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

#### 1. Setup non-root Linux user
> You can reasonably skip this section if you followed `Section 2: Join the Proxmox Host to the Active Directory Domain` of the PVE AD Authentication Guide as you will be able to logon using your AD account: `user@domain.com`

In enterprise environments, direct root logins should be avoided for day-to-day administration. Creating a dedicated administrative user improves auditability, reduces the risk of accidental destructive commands, and enables more controlled access patterns (such as MFA-backed SSH keys, per-user sudo logging, and account-level offboarding).

> **Security**
>
> Using a named, non-root account improves accountability and traceability.  
> Administrative actions can be attributed to an individual user rather than a shared `root` login.
>
> **Trade-off**
>
> This introduces an extra step for privileged actions (`sudo`).  
> However, the operational overhead is minimal compared to the security and audit benefits.  

---

**Install `sudo` (required on Proxmox VE)**  
> Proxmox VE does not install `sudo` by default. If you plan to administer the host using a non-root user, install `sudo` first so the user can perform privileged actions in a controlled and auditable way.  

Install `sudo`:  
```
apt update && apt install -y sudo
```
Verify it is installed:
```
sudo -V
```
---

Create a dedicated administrative user (replace `adminuser` with your preferred name):
```
adduser adminuser
```

Add the user to the `sudo` group to allow administrative access:
```
usermod -aG sudo adminuser
```
> It is not necessary to set `Full Name`, `Room Number`, `Work Phone`, `Home Phone`, or `Other` for the new user.

Verify group membership:

```
id adminuser
```
> OR
```
groups adminuser
```
Test sudo access by switching to the new user and running a privileged command:

```
su - adminuser
```

```
sudo -V
```

> If you are using SSH key-based access, configure the user’s authorized keys under `/home/adminuser/.ssh/authorized_keys` and verify login works before disabling root SSH access.

Optional (recommended for enterprise operations): enforce least privilege by requiring sudo for administrative commands and avoid using `su` to switch to root for routine tasks.

#### 2. Enforce least privilege with `sudo`

In enterprise environments, routine administration should be performed from a named, non-root account with `sudo` privileges rather than switching directly to the `root` user. This supports least privilege, improves accountability, and makes it easier to audit administrative actions.

> **Why this matters**
>
> Using `sudo` creates a clearer separation between normal user activity and privileged actions. It also provides better logging and reduces the risk of accidental destructive commands being run in a fully privileged shell.

---

**Best practice (balanced)**

This approach is recommended for most enterprise environments. It enforces least privilege while preserving controlled break-glass access for recovery workflows.

> The following approaches address privilege escalation in different ways.:
>
> - Restricting `su` controls **who is allowed to switch to the root account**.
> - A `sudo` policy controls **what privileged commands a user can run**, and provides better auditing of administrative actions.
>
> For most enterprise environments, both should be implemented together to prevent privilege bypass and ensure all elevated actions are attributable to a named user.

##### Enforce least privilege through `sudo` policy
> This is unneccessary if already completed as part of creating a non-root user.

Install `sudo` if it is not already installed:

`apt update && apt install -y sudo`

Add the administrative user to the `sudo` group:

`usermod -aG sudo adminuser`

> The `sudo` group grants full administrative privileges by default. If your organization requires more restrictive privileges, define a dedicated sudo policy instead of using full sudo group membership.

---

> If using a non-root user, make sure that you prepend `sudo` to commands 

##### Restrict `su` to a specific group

Create a group for users allowed to use `su`:

```
groupadd suusers
```

Add approved administrative users to the group:

```
usermod -aG suusers adminuser
```

Edit the `su` PAM policy:

```
nano /etc/pam.d/su
```

Add the following line near the top of the file:

```
# This restricts su usage to users in the suusers group
auth required pam_wheel.so group=suusers
```

> This restricts the use of `su` so only members of the `suusers` group can switch to the root account.

---

##### Improve auditing and enforcement (enterprise-ready)

Enable logging for sudo activity:

```
visudo
```

Add the following lines:

```
Defaults        logfile="/var/log/sudo.log"
```

> This creates a dedicated sudo audit log for administrative actions.

Optional (more strict): shorten the sudo timestamp window so users re-authenticate more frequently:

```
Defaults        timestamp_timeout=5
```

> This reduces the chance of unattended terminals retaining elevated privileges.  
> The default sudo timeout is commonly 5–15 minutes depending on distribution and organizational hardening policies. Each successful sudo command refreshes the timeout window, extending the time before the next password prompt. Always verify your effective timeout by checking timestamp_timeout in /etc/sudoers or using `visudo`.


---

**High-control environments**

This approach is intended for highly regulated environments where root switching must be tightly controlled or eliminated. Only use this if you have validated your emergency recovery process and have console access (or out-of-band management such as iDRAC/iLO).

##### Disable `su` entirely (stronger control)

Remove the SUID bit from `/bin/su` to prevent any user from using `su`:

```
chmod u-s /bin/su
```

Verify the SUID bit is removed:

```
ls -l /bin/su
```

> This forces all privileged activity through `sudo` and prevents root switching entirely. Ensure you have at least one working sudo-enabled account and out-of-band access before doing this.

---

##### Enforce least privilege through restricted `sudo` rules

Instead of granting full root access, restrict what commands the administrative user may run:

```
visudo -f /etc/sudoers.d/adminuser
```

Example policy (adjust to your operational needs):

```
adminuser ALL=(root) /usr/bin/systemctl *, /usr/bin/journalctl *, /usr/bin/apt *, /usr/bin/apt-get *
```

> Restricted sudo rules provide true least privilege, but require ongoing maintenance as operational requirements evolve.

---

##### Improve auditing and enforcement (high-control)

Enable dedicated logging:

```
visudo
```

Add:

```
Defaults        logfile="/var/log/sudo.log"
```

Optional: force sudo to require a password every time:

```
Defaults        timestamp_timeout=0
```

> This prevents reuse of cached sudo credentials and ensures every privileged action is deliberate.

Optional (advanced): enable sudo I/O logging (where supported):

```
Defaults        log_input,log_output
```

> This provides stronger auditability but can generate significant log volume. Ensure log forwarding and storage retention policies are in place.

---

**Operational guidance**

- Use `sudo <command>` for one-off administrative actions instead of switching to root.
- Avoid `su -` for routine work, even in balanced environments.
- Maintain a documented break-glass process (console access, emergency credentials, and recovery procedures) for root-level recovery scenarios.

#### 3. Configure SSH Access and Settings

Configuring SSH correctly is critical in enterprise environments.   
The goal is to enforce strong authentication, reduce attack surface, and ensure cluster operations are not disrupted.

> Make sure to align all SSH settings with your organization’s security requirements and access policies.
>
> It is generally **not recommended** to set `PermitRootLogin no` on Proxmox clusters unless you have validated that all cluster operations, automation, and break-glass procedures are still supported. Proxmox clusters use SSH keys for node-to-node communication, and overly restrictive root login policies can complicate recovery workflows.
>
> Always check the bottom of the SSH config file for duplicate or overriding settings (for example, an additional `PasswordAuthentication yes` entry later in the file).

---

##### Edit the SSH daemon configuration

```
nano /etc/ssh/sshd_config
```

> Replace `nano` with `vi`, `vim`, or another editor if you prefer a different editor.

---

##### Find, enable, and set the following values

```
Port 22
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
PrintLastLog yes
```

> `PermitRootLogin prohibit-password` allows root login only using SSH keys (no password authentication).  
> This supports cluster node communication while still blocking password-based root access.  
> Additional it provides a way to still access the root user via SSH in a break-glass scenario.  

---

##### Validate and apply the configuration

Test the SSH configuration for syntax errors:

`sshd -t`

Reload SSH to apply changes without dropping active sessions:

`systemctl reload ssh`

> Use `reload` instead of `restart` to reduce the chance of disconnecting active administrative sessions.

---

#### Generate a private/public key pair

For enterprise environments, key-based authentication should be mandatory for administrative access.

> We recommend using Bitvise SSH Client’s **Client Key Manager** for key generation and management.
>
> Standardize on the `Ed25519` algorithm unless your organization requires otherwise.
>
> Use a passphrase for private keys to reduce risk if a key file is exposed.

If you are not using Bitvise, generate an Ed25519 key pair from the Proxmox host (or your admin workstation):

```
ssh-keygen -t ed25519 -a 64 -C "adminuser"
```
> The `-a 64` option increases key derivation rounds, making passphrase brute-forcing significantly harder.
> Replace `adminuser` with your username. Include your domain name if using an AD user.

---

#### Export an OpenSSH-format public key

If you are using Bitvise, you can export the public key in OpenSSH format using the Client Key Manager.

If you generated the key using `ssh-keygen`, the OpenSSH public key is already available as:
```
~/.ssh/id_ed25519.pub
```
You can display it for copy/paste with:
```
cat ~/.ssh/id_ed25519.pub
```

---

#### Add the public key to the administrative user

Place the public key into the target user’s authorized keys file (Make sure to replace `adminuser` with your username):

```
mkdir -p /home/adminuser/.ssh
```
```
chmod 700 /home/adminuser/.ssh
```
```
nano /home/adminuser/.ssh/authorized_keys
```
Paste the public key and save.   
> Example key: `ssh-ed25519 JJJJB4PzaA2lZDI1HTY8JAJJILST+ad7U1ShZ+e04Kg4t1VC1wUnyCfbqrSxnofHRSqe`    
> The above key is fake, do not use it.  

Then set permissions:
```
chmod 600 /home/adminuser/.ssh/authorized_keys
```
```
chown -R adminuser:adminuser /home/adminuser/.ssh
```
> Correct permissions are required or SSH will ignore the key for security reasons.

---

#### Verify key-based login

From your workstation:

```
ssh adminuser@<proxmox-hostname-or-ip>
```
> Or use your chosen SSH tool such as PuTTY, Bitvise, mRemoteNG, etc.  

Once verified, ensure that password authentication remains disabled (`PasswordAuthentication no`) and that only authorized users can access SSH.

---

### III. Proxmox Web UI Permissions and Roles Best Practices

This section focuses on best practices for managing authorization within the Proxmox Web GUI, independent of the underlying authentication source. It applies equally to environments using local Proxmox authentication as well as enterprise environments integrated with Active Directory (or another external identity provider).

Topics include:

- Understanding Proxmox roles and privilege separation
- Assigning permissions using Proxmox objects and path-based scope
- Avoiding overuse of the `PVEAdmin` role and the `Administrator` role
- Creating custom roles when default roles are overly permissive
- Ensuring permissions align with operational responsibilities
- Using group-based permission assignments instead of individual users

Authentication backends (such as Active Directory) are assumed to be configured separately and are not covered in this section. This section focuses on how to apply permissions correctly once users and groups exist in Proxmox.

---

#### 1. Understanding Proxmox roles and privilege separation

**Roles**

A role is simply a list of privileges. Proxmox VE comes with a number of predefined roles, which satisfy most requirements.   
You can view the full set of predefined roles in the Proxmox GUI.

Predefined roles include:

- `Administrator`: has full privileges
- `NoAccess`: has no privileges (used to forbid access)
- `PVEAdmin`: can do most tasks, but has no rights to modify system settings (`Sys.PowerMgmt`, `Sys.Modify`, `Realm.Allocate`) or permissions (`Permissions.Modify`)
- `PVEAuditor`: has read only access
- `PVEDatastoreAdmin`: create and allocate backup space and templates
- `PVEDatastoreUser`: allocate backup space and view storage
- `PVEMappingAdmin`: manage resource mappings
- `PVEMappingUser`: view and use resource mappings
- `PVEPoolAdmin`: allocate pools
- `PVEPoolUser`: view pools
- `PVESDNAdmin`: manage SDN configuration
- `PVESDNUser`: access to bridges/vnets
- `PVESysAdmin`: audit, system console and system logs
- `PVETemplateUser`: view and clone templates
- `PVEUserAdmin`: manage users
- `PVEVMAdmin`: fully administer VMs
- `PVEVMUser`: view, backup, configure CD-ROM, VM console, VM power management

---

#### 2. Assign permissions using objects and paths

**Objects and Paths**

Access permissions are assigned to objects, such as virtual machines, storages, or resource pools. Proxmox uses file system-like paths to address these objects. These paths form a natural tree, and permissions at higher levels (shorter paths) can optionally be propagated down within this hierarchy.

Paths can also be templated. When an API call requires permissions on a templated path, the path may contain references to parameters of the API call. These references are specified in curly braces. Some parameters are implicitly taken from the API call’s URI. For instance, the permission path `/nodes/{node}` when calling `/nodes/mynode/status` requires permissions on `/nodes/mynode`, while the path `{path}` in a PUT request to `/access/acl` refers to the method’s path parameter.

Some common examples:

- `/`: The root level, granting access to all objects in the datacenter. Permissions here affect the entire cluster.
- `/nodes`: Access to all Proxmox VE nodes in the cluster.
- `/nodes/{node}`: Access to Proxmox VE server machines
- `/vms`: Access to all VMs
- `/vms/{vmid}`: Access to a specific VM
- `/storage/{storeid}`: Access to a specific storage
- `/pool/{poolname}`: Access to resources contained in a specific pool
- `/access/groups`: Group administration
- `/access/realms/{realmid}`: Administrative access to realms

> Best practice is to assign permissions at the lowest reasonable path to limit scope and reduce unintended inherited access.

**Inheritance**

Object paths form a file system like tree, and permissions can be inherited by objects down that tree (the propagate flag is set by default). Proxmox VE uses the following inheritance rules:

- Permissions for individual users always replace group permissions.
- Permissions for groups apply when the user is member of that group.
- Permissions on deeper levels replace those inherited from an upper level.
- `NoAccess` cancels all other roles on a given path.
- Privilege separated tokens can never have permissions on any given path that their associated user does not have.

> Inheritance is powerful but can lead to unintended access if permissions are granted too high in the tree. Prefer narrow paths and review the propagate flag carefully.

---

#### 3. Avoid overuse of `PVEAdmin` and `Administrator`

Both `PVEAdmin` and `Administrator` are high-privilege roles and should be reserved for trusted infrastructure administrators.

- `Administrator` has full privileges and should be tightly restricted.
- `PVEAdmin` is still highly privileged and can perform most operational tasks, but it cannot modify certain system settings or permissions.

Use cases for limited roles:

- VM operators should typically use `PVEVMUser` scoped to the appropriate VM paths or pools. 
- Reserve `PVEVMAdmin` for teams responsible for full VM lifecycle management, including hardware configuration changes.
- Auditors and support teams should use `PVEAuditor` or a restricted read-only role.

> Avoid granting cluster-wide `Administrator` or `PVEAdmin` access unless it is operationally required and documented.

---

#### 4. Create custom roles when defaults are overly permissive

If built-in roles do not map cleanly to your operational model, create custom roles that match enterprise responsibilities. Custom roles reduce the need for broad administrative access and support least privilege without relying on individual exceptions.

You can add new roles via the GUI or the command line.

**Create a role via the GUI**

- Datacenter → Permissions → Roles → Create
- Set a role name
- Select privileges from the **Privileges** drop-down menu

**Create a role via the command line**

Use the `pveum` CLI tool, for example:

`pveum role add VM_Power-only --privs "VM.PowerMgmt VM.Console"`

`pveum role add Sys_Power-only --privs "Sys.PowerMgmt Sys.Console"`

> Roles starting with `PVE` are always builtin. Custom roles are not allowed to use this reserved prefix.

**Privileges**

A privilege is the right to perform a specific action. To simplify management, lists of privileges are grouped into roles, which can then be used in the permission table. Privileges cannot be directly assigned to users and paths without being part of a role.

Node / System related privileges:

- `Group.Allocate`: create/modify/remove groups
- `Mapping.Audit`: view resource mappings
- `Mapping.Modify`: manage resource mappings
- `Mapping.Use`: use resource mappings
- `Permissions.Modify`: modify access permissions
- `Pool.Allocate`: create/modify/remove a pool
- `Pool.Audit`: view a pool
- `Realm.AllocateUser`: assign user to a realm
- `Realm.Allocate`: create/modify/remove authentication realms
- `SDN.Allocate`: manage SDN configuration
- `SDN.Audit`: view SDN configuration
- `Sys.Audit`: view node status/config, Corosync cluster config, and HA config
- `Sys.Console`: console access to node
- `Sys.Incoming`: allow incoming data streams from other clusters (experimental)
- `Sys.Modify`: create/modify/remove node network parameters
- `Sys.PowerMgmt`: node power management (start, stop, reset, shutdown, …)
- `Sys.Syslog`: view syslog
- `User.Modify`: create/modify/remove user access and details

Virtual machine related privileges:

- `SDN.Use`: access SDN vnets and local network bridges
- `VM.Allocate`: create/remove VM on a server
- `VM.Audit`: view VM config
- `VM.Backup`: backup/restore VMs
- `VM.Clone`: clone/copy a VM
- `VM.Config.CDROM`: eject/change CD-ROM
- `VM.Config.CPU`: modify CPU settings
- `VM.Config.Cloudinit`: modify Cloud-init parameters
- `VM.Config.Disk`: add/modify/remove disks
- `VM.Config.HWType`: modify emulated hardware types
- `VM.Config.Memory`: modify memory settings
- `VM.Config.Network`: add/modify/remove network devices
- `VM.Config.Options`: modify any other VM configuration
- `VM.Console`: console access to VM
- `VM.GuestAgent.Audit`: issue informational QEMU guest agent commands
- `VM.GuestAgent.FileRead`: read files from the guest via QEMU guest agent
- `VM.GuestAgent.FileSystemMgmt`: freeze/thaw/trim file systems via QEMU guest agent
- `VM.GuestAgent.FileWrite`: write files in the guest via QEMU guest agent
- `VM.GuestAgent.Unrestricted`: issue arbitrary QEMU guest agent commands
- `VM.Migrate`: migrate VM to alternate server on cluster
- `VM.PowerMgmt`: power management (start, stop, reset, shutdown, …)
- `VM.Replicate`: configure and run guest replication
- `VM.Snapshot.Rollback`: rollback VM to one of its snapshots
- `VM.Snapshot`: create/delete VM snapshots

Storage related privileges:

- `Datastore.Allocate`: create/modify/remove a datastore and delete volumes
- `Datastore.AllocateSpace`: allocate space on a datastore
- `Datastore.AllocateTemplate`: allocate/upload templates and ISO images
- `Datastore.Audit`: view/browse a datastore

Warning: Both `Permissions.Modify` and `Sys.Modify` should be handled with care, as they allow modifying aspects of the system and its configuration that are dangerous or sensitive.  
Warning: Carefully read the section about inheritance to understand how assigned roles (and their privileges) are propagated along the ACL tree.

---

#### 5. Ensure permissions align with operational responsibilities

Design groups and roles around the teams who perform work:

- Infrastructure administrators
- VM operators / application teams
- Backup operators
- Security / audit teams
- Helpdesk / support roles

> Permissions should support least privilege and separation-of-duties. If a group must have broad permissions, document why and review the assignment periodically.

---

#### 6. Recommended break-glass access (enterprise)

Even in AD environments, maintain a minimal, documented break-glass access path:

- A local Proxmox user not dependent on AD availability
- Stored securely (password vault), rotated regularly, and monitored
- Used only for emergencies and tested periodically

> Break-glass accounts should not be used for routine operations and should have clear ownership and escalation procedures.

---

#### 7. Group-based access model (recommended)

Proxmox permissions should be assigned to **groups**, not individual users. Group-based RBAC supports least privilege, simplifies onboarding/offboarding, and ensures consistent access control across clusters.

> Assigning permissions to individual users makes access control harder to audit, increases the chance of privilege drift, and complicates operational handoffs.

**A. Non-AD environments (local Proxmox users and groups)**  
> Skip to `B. AD environments (recommended for enterprise)` below if using an AD integrated environment.

In environments not integrated with Active Directory, use Proxmox “local” users and groups to implement consistent RBAC.

**Create a local Proxmox user (if not already done)**

If you have not yet created a dedicated local Proxmox user (recommended), create one first.  
This improves accountability and avoids relying exclusively on the built-in `root@pam` account for day-to-day management.

For clustered environments, prefer using the **Proxmox VE authentication server** (`pve` realm) instead of `pam`. `pve` users are stored and managed within Proxmox and are available across all nodes in the cluster, while `pam` users are local to each individual host.

- Datacenter → Permissions → Users → Add
- Realm: `pve` (Proxmox VE authentication server)
- Username: e.g., `proxmoxadmin`
- Ensure strong password requirements and enable MFA where supported

> In enterprise environments, local accounts should be limited in number, documented, and protected by strong password and MFA policies. Consider reserving the `root@pam` account as a break-glass or emergency-only account.

**Create local Proxmox groups**

Create at least two baseline groups:

- `ProxmoxAdmins`
- `ProxmoxUsers`

> These group names are examples. Use naming that matches your organization’s standards.

Create the groups in the Proxmox Web UI:

- Datacenter → Permissions → Groups → Create

**Add local users to groups**

- Datacenter → Permissions → Users
- Double-click the user entry (or select the user and click **Edit**) → Assign the user to the appropriate group (`ProxmoxAdmins` or `ProxmoxUsers`)

**Assign roles to groups (not users)**

- Datacenter → Permissions → Add → Group Permission
- Select the group and assign a Proxmox role at the appropriate path
> Repeat for all groups

---

**B. AD environments (recommended for enterprise)**

In AD-integrated environments, prefer using AD-synced groups rather than creating and managing local Proxmox groups.

**Use AD-synced security groups**

Create and manage groups in Active Directory such as:

- `ProxmoxAdmins`
- `ProxmoxUsers`
- `ProxmoxReadOnly`
- `ProxmoxVMOperators`

Once available to Proxmox through your authentication backend, assign these groups permissions in Proxmox.

> AD group ownership and membership should align with your organization’s access control governance (ticketing approvals, role-based membership, and offboarding workflows).

**Assign roles to AD groups in Proxmox**

- Datacenter → Permissions → Add → Group Permission
- Select the AD group and assign a Proxmox role at the appropriate path
> Repeat for all groups

> Using AD groups ensures access is automatically revoked when accounts are disabled or removed from the AD group, reducing long-term risk.

**Avoid mixing local and AD models unless required**

If AD is configured, avoid building parallel “local” RBAC models unless you require break-glass local accounts. If break-glass accounts exist, they should be limited, monitored, and documented.

---

### IV. Virtual Machine CPU and Memory Configuration Best Practices

This section provides guidance for CPU and memory settings that apply to the majority of virtual machine workloads. The goal is to balance performance, compatibility, and operational flexibility (especially live migration and cluster scalability) in enterprise environments.

Topics include:

- Recommended CPU types for most workloads and when to avoid host passthrough
- Live migration considerations related to CPU selection
- When and why to enable NUMA
- Memory allocation strategies, including ballooning considerations
- Overcommit guidance and when to avoid aggressive memory overcommit
- High-level discussion of hugepages and when they may be appropriate

---

#### 1. Recommended CPU types and when to avoid host passthrough

For most enterprise workloads, use a **cluster-compatible CPU type** rather than `host` passthrough. This improves compatibility, reduces operational risk, and enables smooth live migration across nodes by ensuring a consistent, predictable CPU feature set across your cluster.

**Recommended CPU model approach**

- Prefer the **`x86-64-v2` / `x86-64-v2-AES` / `x86-64-v3` / `x86-64-v4`** family where supported
- Use **`qemu64`** only for legacy compatibility requirements
- Use **`kvm64`** only when required for older guest OSes or strict compatibility constraints
- Avoid **`host`** unless you have a specific workload requirement and have validated migration constraints

> Using a consistent CPU type across the cluster reduces migration failures and prevents subtle performance or feature mismatches across nodes.

---

**Understanding the `x86-64-v2` / `v3` / `v4` CPU model levels**

The `x86-64-v*` models are standardized CPU feature baselines designed to provide a balance of performance and portability. Each version represents a newer baseline with more modern instruction set requirements.

- **`x86-64-v2`**
  - Broad compatibility across modern server hardware
  - Adds commonly expected instruction sets beyond the original x86-64 baseline
  - Suitable as a cluster-default when hardware generations vary

- **`x86-64-v2-AES`**
  - Same baseline as `x86-64-v2`, but explicitly ensures AES-NI support
  - Useful for workloads where encryption performance matters (TLS-heavy services, VPNs, storage encryption)
  - Recommended when you want a broadly compatible baseline but do not want to lose AES acceleration

- **`x86-64-v3`**
  - Requires additional modern instruction sets (notably AVX/AVX2 class capabilities)
  - Can improve performance in compute-heavy workloads (compression, encryption, analytics, certain databases)
  - Only use if all hosts in the cluster meet the feature requirements

- **`x86-64-v4`**
  - The most modern baseline (includes newer vector instruction capabilities such as AVX-512 class features on capable hardware)
  - Can provide performance improvements for specialized workloads that benefit from these instructions
  - Not recommended as a default unless your cluster hardware is highly consistent and known to support it

> In general:  
> - `x86-64-v2` is the safest broad enterprise default when hardware generations vary.  
> - `x86-64-v2-AES` is a strong default when you want the broad compatibility of `v2` but also want to ensure AES-NI is consistently exposed for encryption-heavy workloads.  
> - `x86-64-v3` is a good option for clusters with uniformly modern CPUs (and can improve performance for compute-heavy workloads).  
> - `x86-64-v4` should be reserved for specialized workloads on very consistent, modern hardware, since it requires newer instruction sets that are not universally available.  


---

**Legacy compatibility models**

- **`qemu64`**
  - Very generic CPU model with high compatibility
  - Often used when maximum portability is required across diverse hardware
  - Can reduce performance by hiding modern CPU features

- **`kvm64`**
  - Similar intent to `qemu64`, but may expose slightly different virtualization features depending on environment
  - May be required for older OSes or extremely compatibility-sensitive workloads
  - Not recommended for modern clusters unless required

> If you have old guests or mixed/unknown CPU hardware, `qemu64` is the most portable option, but it may sacrifice performance and modern CPU acceleration features.

---

**When to avoid or use `host` CPU type**

Using the `host` CPU type in Proxmox VE gives your virtual machines (VMs) the best performance by exposing **all of the host’s CPU features (flags)** directly to the guest. This can improve performance for workloads that benefit from specific CPU accelerations (encryption, compression, SIMD/vector operations, databases, HPC workloads).

However, `host` passthrough significantly limits operational flexibility:

- Live migration may fail if the destination host does not support the exact same CPU flags
- Hardware refreshes or mixed CPU generations can cause VM start failures on other nodes
- Cluster upgrades become more complex because CPU feature sets must remain compatible

> Use `host` only when you have a measured workload requirement, uniform CPU hardware across the cluster, and you have validated that your live migration and recovery workflows still function as expected.

---

**Modern vs legacy CPU examples (helpful when selecting a cluster CPU type)**

The `x86-64-v2` and `x86-64-v2-AES` CPU profiles are designed as broadly compatible baselines and are often the safest default for enterprise clusters, especially when hardware generations vary. The newer `x86-64-v3` and `x86-64-v4` profiles provide additional modern instruction sets and can improve performance, but they work best only when cluster CPU hardware is consistently modern and uniform. If your environment includes older servers or mixed CPU generations, standardizing on `x86-64-v2` (or in extreme legacy cases `qemu64`) helps maintain compatibility and predictable migration behavior.


**Examples of modern enterprise-class CPUs (typically safe for `x86-64-v2` and often `x86-64-v3`)**

- **Intel Xeon Scalable (Skylake-SP and newer):** 2017+ (Bronze/Silver/Gold/Platinum)  
- **Intel Xeon Scalable (Ice Lake / Sapphire Rapids):** 2021+ / 2023+ (strong candidates for `x86-64-v3` and in some cases `x86-64-v4`)  
- **AMD EPYC (Naples / Rome / Milan / Genoa):** 2017+ (solid `v2`, commonly `v3` in homogeneous clusters)  
- **AMD Threadripper / Threadripper Pro (enterprise workstation tiers):** 2017+ (commonly supports `v3`; treat as “server-like” only if deployed consistently across nodes)

**What is typically safe for `x86-64-v4`?**

`x86-64-v4` is the most demanding standardized CPU baseline and generally maps to **AVX-512 class** capabilities.  
It is only a safe cluster default when **every node** supports the required instruction sets.

- **Most commonly safe:** **Intel Xeon Scalable 3rd Gen (Ice Lake-SP) and newer:** 2021+  
- **Strong candidates:** **Intel Xeon Scalable 4th Gen (Sapphire Rapids):** 2023+  
- **Sometimes safe (validate per model):** **Intel Xeon Scalable 2nd Gen (Cascade Lake-SP):** 2019 (many SKUs support AVX-512, but cluster consistency and BIOS exposure still matter)

`x86-64-v4` is **generally not recommended** for:

- **AMD EPYC / AMD Threadripper / AMD-based clusters** (AVX-512 support differs and is not a consistent baseline)
- **Mixed Intel + AMD clusters**
- **Clusters with mixed CPU generations or stepping differences**

> Treat `x86-64-v4` as a specialized baseline for clusters built on uniform Intel AVX-512 capable hardware. For most enterprise clusters, `x86-64-v2` / `x86-64-v2-AES` is the safest default, and `x86-64-v3` is a good choice when hardware is consistently modern.

**Examples of legacy CPUs (often require conservative baselines)**

- **AMD Opteron (many generations):** typically 2010–2016 era (often best suited for `kvm64` or `qemu64` depending on model and compatibility needs)  
- **Intel Xeon 55xx / 56xx (Nehalem/Westmere):** 2009–2011 (legacy baseline; frequently requires older CPU models)  
- **Intel Xeon E5 v1/v2 (Sandy Bridge / Ivy Bridge):** 2012–2013 (may still work with `x86-64-v2`, but cluster consistency must be validated)  
- **Older mixed hardware clusters (pre-2014):** strongly consider `x86-64-v2` or `qemu64` for predictable migration behavior  

As a general rule, **servers from ~2014 or earlier** should be treated as potentially legacy for migration planning, and **servers from 2017+** are typically modern enough for `x86-64-v2` and often `x86-64-v3` if the cluster is consistent.

---

#### 2. Live migration and CPU compatibility considerations

Live migration requires CPU feature compatibility between the source and destination hosts. Even small differences between CPU generations can break migrations when using passthrough CPU types.

Best practices:

- Standardize CPU type at the cluster level
- Avoid mixing CPU families or generations when possible
- When mixing is unavoidable, choose the lowest common CPU model that supports required features
- Validate migration behavior during cluster expansion or hardware refresh

> CPU incompatibility is one of the most common root causes of migration failures in mixed-hardware clusters.
>
> In mixed-hardware environments or when upgrading to newer hardware, it may be necessary to temporarily adjust a VM’s CPU type away from `host` to a more compatible profile (such as `x86-64-v2-AES` or `x86-64-v3`) to enable a successful migration or restore. This change should be performed during a controlled maintenance window and validated to ensure the VM boots and operates correctly under the temporary CPU profile.

---

#### 3. When and why to enable NUMA

NUMA (Non-Uniform Memory Access) awareness is relevant for larger VMs and systems with multiple CPU sockets.  
Enabling NUMA helps the guest OS align CPU scheduling and memory locality, reducing cross-socket memory access penalties.

General guidance:

- For small to medium VMs (low vCPU count, moderate memory), NUMA usually provides little benefit
- For large VMs (high vCPU count and large RAM allocations), NUMA can improve performance and reduce latency

NUMA is most relevant when:

- VMs use large memory allocations (tens of GB or more)
- VMs have high core counts (typically 8+ vCPUs, more often 16+)
- workloads are latency sensitive (databases, analytics, high-throughput services)

> NUMA tuning is most effective when host hardware is properly aligned and the VM is sized to fit within a NUMA node. Poor NUMA alignment can reduce performance rather than improve it.

Key Benefit:  
- Avoids Remote Access: Ensures a VM's vCPUs primarily use memory directly attached to their physical CPU socket, avoiding slow access over the interconnect bus to memory on another socket. 

---

#### 4. Memory allocation strategies and ballooning considerations

Memory allocation strategy is a key performance and stability decision.   
The best approach depends on workload predictability and operational requirements.

Recommended guidance:

- Use fixed memory allocation for most production workloads that require predictable performance
- Use ballooning cautiously and only when you have capacity planning and monitoring in place

**Ballooning**

Ballooning allows the hypervisor to reclaim unused memory from VMs under pressure and redistribute it to other VMs.

Benefits:

- Helps increase consolidation density
- Provides flexibility when workloads are predictable and monitored

Risks:

- Guests may behave unpredictably under reclaimed memory
- Can cause latency spikes, cache eviction, and degraded application performance
- Some workloads perform poorly when memory availability fluctuates

> Ballooning should not be treated as a substitute for proper capacity planning. If a VM is business-critical, allocate the memory it needs and avoid frequent reclamation events.

---

#### 5. Overcommit guidance and when to avoid aggressive overcommit

Memory overcommit enables higher VM density, but it introduces operational risk. Overcommit should be deliberate and aligned with monitoring, workload predictability, and recovery expectations.

General enterprise guidance:

- Light overcommit can be acceptable if workloads are stable and predictable
- Avoid aggressive overcommit unless you have strong monitoring and a clear failure handling strategy
- Avoid overcommit for latency-sensitive workloads (databases, critical service tiers, real-time workloads)

Signs you should reduce overcommit:

- frequent swap activity on the host
- recurring memory pressure events
- ballooning reclamation in critical VMs
- kernel OOM events or service instability
- VM performance anomalies correlated with host memory pressure

> Overcommit failures are not graceful. If the host runs out of reclaimable memory, the kernel may invoke the OOM killer and terminate processes, including Proxmox services or VM-related workloads.

---

## V. Network Interface Best Practices

This section defines **baseline network interface best practices** for Proxmox VE hosts and virtual machines. These practices are intended to provide predictable behavior, broad compatibility, and stable operations across the majority of environments. They focus on safe defaults, supported configurations, and patterns that reduce operational risk rather than performance tuning.

The guidance below is divided into host-level and VM-level considerations. All items are expected to be applicable to most Proxmox VE deployments unless a clear, documented reason exists to deviate.

---

### V.A. Host-Level Network Interface Best Practices

These practices apply to almost all Proxmox VE host deployments.

**Topics include:**

- Using native Linux bridges (`vmbrX`) for virtual machine networking
- Avoiding ad-hoc or unsupported networking stacks unless explicitly required
- Separating management, cluster, and storage traffic where possible
- Maintaining supported and up-to-date NIC drivers and firmware
- Ensuring consistent MTU configuration across hosts, switches, and connected networks
- Verifying correct link speed and duplex negotiation

#### 1. Use Linux bridges for VM networking

Use native Linux bridges (`vmbrX`) for VM connectivity.  
Avoid custom or non-standard networking approaches unless required by design.  
Align with Proxmox-supported and well-documented networking models.

**Why:**  
Predictable behavior, broad support, and simpler troubleshooting.

---

#### 2. Avoid overloading the management interface

Separate management traffic from heavy VM traffic when possible.  
Use dedicated interfaces or VLANs for:
- Proxmox management
- Cluster communication
- Storage traffic (Ceph, NFS, iSCSI)

**Why:**  
Prevents management instability during traffic spikes.

---

#### 3. Keep NIC drivers and firmware up to date

Use vendor-supported NIC firmware and drivers.  
Coordinate updates with Proxmox VE and kernel release cycles.

**Why:**  
Addresses stability issues, performance defects, and known vulnerabilities.

---

#### 4. Use consistent MTU settings across network paths

Ensure MTU values are consistent end-to-end (host, switch, storage, VM).  
Avoid partial or inconsistent jumbo frame configurations.

**Why:**  
MTU mismatches cause packet loss and difficult-to-diagnose performance issues.

---

#### 5. Validate link speed and duplex

Ensure interfaces negotiate expected link speed and duplex settings.  
Investigate auto-negotiation issues rather than forcing static values.

**Why:**  
Silent mis-negotiation can severely degrade throughput and increase latency.

---

### V.B. VM-Level Network Interface Best Practices

These practices apply to most virtual machine workloads regardless of performance profile.

**Topics include:**

- Selecting supported virtual NIC models
- Minimizing unnecessary virtual network complexity
- Maintaining predictable addressing and migration behavior

---

#### 6. Use VirtIO network interfaces for most workloads

Default to VirtIO network adapters for Linux and modern Windows guests.  
Install and maintain VirtIO drivers for Windows VMs.

**Why:**  
Provides the best balance of performance, stability, and long-term support.

---

#### 7. Assign appropriate vNIC count and bandwidth

Avoid assigning unnecessary multiple network interfaces.  
Prefer VLAN segmentation or traffic shaping over additional vNICs where possible.

**Why:**  
Reduces guest complexity and improves operational predictability.

---

#### 8. Avoid unnecessary promiscuous mode

Enable promiscuous mode only for workloads that explicitly require it.  
Document exceptions when enabled.

**Why:**  
Reduces attack surface and unintended traffic exposure.

---

#### 9. Use stable MAC addressing

Allow Proxmox VE to manage MAC addresses or document static assignments.  
Avoid frequent or unnecessary MAC address changes.

**Why:**  
Prevents DHCP, firewall, and monitoring disruptions.

---

#### 10. Validate live migration with network dependencies

Test live migration for VMs with network-specific requirements.  
Ensure bridge and VLAN consistency across all cluster nodes.

**Why:**  
Network mismatches are a common cause of live migration failures.

---

### What Is Intentionally Not Included

The following items are **not** baseline best practices and are covered in the Optional Practices section:

- TCP BBR and TCP Fast Open
- NIC queue length tuning
- Network-related sysctl performance tuning
- Disabling IPv6
- Jumbo frames (unless validated end-to-end)
- SR-IOV configurations

---

## VI. Storage and Disk Configuration Best Practices

This section defines **baseline storage and disk configuration best practices** for Proxmox VE virtual machines. The guidance focuses on safe, broadly applicable defaults that promote stability, predictable performance, and data integrity across the majority of environments. These practices are expected to be suitable for most workloads and should generally be implemented unless there is a documented reason to diverge.

The intent of this section is to establish a reliable storage foundation rather than to optimize for specialized performance characteristics.

Topics include:

- Recommended default disk controller selection for most workloads
- Baseline use of VirtIO-SCSI and when it is preferred over VirtIO block
- Default asynchronous I/O mode selections that balance performance and stability
- Disk cache mode defaults and associated safety considerations
- SSD emulation defaults and when enabling it is generally appropriate
- Discard and TRIM usage guidelines that are safe for most storage backends
- Log Rotation Policy Tuning

---

#### 1. Default Disk Controller Selection (VirtIO-SCSI)

**Basic Instructions:**
- Use **VirtIO-SCSI** as the default disk controller for most virtual machines.
- Prefer a single VirtIO-SCSI controller unless there is a clear need for separation.
- Ensure guest OS support and drivers are installed (especially for Windows).

**Performance Trade-offs:**
- VirtIO-SCSI provides better scalability and feature support than VirtIO block.
- Slightly more overhead than VirtIO block in some microbenchmarks, but negligible in real-world workloads.
- Enables advanced features such as discard/TRIM and multi-queue support.

---

#### 2. Use of Single vs. Multiple SCSI Controllers

**Basic Instructions:**
- Use a **single SCSI controller** for most VMs.
- Add additional SCSI controllers only when device limits or isolation requirements justify it.

**Performance Trade-offs:**
- Multiple controllers increase configuration complexity.
- May provide marginal benefits for very disk-heavy workloads, but usually unnecessary.
- Single-controller configurations are simpler to manage and migrate.

---

#### 3. Default Asynchronous I/O Mode Selection

**Basic Instructions:**
- Use the **default asynchronous I/O mode** provided by Proxmox VE for the selected storage backend.
- Avoid overriding I/O modes unless addressing a specific, measured issue.

**Performance Trade-offs:**
- Native and io_uring modes can improve performance on modern kernels and storage, but may expose edge cases.
- Threaded I/O offers compatibility and stability but may have higher CPU overhead.
- Defaults are chosen to balance performance and reliability across environments.

---

#### 4. Disk Cache Mode Defaults

**Basic Instructions:**
- Use the **default cache mode** recommended by Proxmox VE for the storage backend.
- Avoid write-back caching unless data loss risks are fully understood and mitigated.

**Performance Trade-offs:**
- Safer cache modes reduce risk of data corruption during host crashes or power loss.
- Write-back caching can improve performance but significantly increases risk without proper power protection.
- Defaults favor data integrity over maximum throughput.

---

#### 5. SSD Emulation Defaults

**Basic Instructions:**
- Enable **SSD emulation** only when the underlying storage is SSD-based and guests can benefit from it.
- Leave disabled when backing storage is rotational or behavior is unknown.

**Performance Trade-offs:**
- SSD emulation can improve guest-side I/O scheduling and behavior.
- Provides no benefit on non-SSD storage and may mislead the guest OS.
- Generally low risk when accurately reflecting underlying storage characteristics.

---

#### 6. Discard and TRIM Support

**Basic Instructions:**
- Enable discard/TRIM only when the storage backend explicitly supports it safely.
- Verify compatibility with the storage layer before enabling.

**Performance Trade-offs:**
- Can improve long-term space efficiency and SSD performance.
- May introduce overhead or latency on some storage backends.
- Incorrect use can negatively impact performance or storage stability.

---

#### 7. Tweak logrotate so logs do not slowly eat your root disk

This is one of the most underrated “set it and forget it” fixes. Proxmox hosts can accumulate logs over time, and if log rotation policies are too permissive (or not tuned at all), `/var/log` can quietly grow until it consumes the root filesystem. This can cause unexpected service failures, inability to write critical logs, and in worst cases prevent the host from functioning normally.

> **Performance**
>
> Proper log rotation reduces disk usage growth and prevents log-related write amplification. It also helps keep log searches and troubleshooting manageable by ensuring logs are compressed and retained for a predictable window.
>
> **Trade-off**
>
> Aggressive rotation reduces historical retention. If you require longer-term log history, forward logs to a centralized logging system (syslog, SIEM, or log aggregation platform) rather than retaining extensive history on the hypervisor itself.

The script below backs up your existing config and replaces it with a clean daily rotation policy:

```
#!/usr/bin/env bash
# Configure logrotate for Proxmox
set -euo pipefail

ROTATE_DAYS="${ROTATE_DAYS:-14}"

if [[ "${EUID:-$(id -u)}" -ne 0 ]]; then
  echo "ERROR: Please run as root (use sudo)." >&2
  exit 1
fi

# Backup existing config
cp /etc/logrotate.conf "/etc/logrotate.conf.bak.$(date +%F_%H%M%S)"

# Create new config
cat > /etc/logrotate.conf <<EOF
# Proxmox logrotate configuration
daily
rotate ${ROTATE_DAYS}
create
compress
delaycompress
dateext
missingok
notifempty
sharedscripts

include /etc/logrotate.d

/var/log/wtmp {
    missingok
    monthly
    create 0664 root utmp
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0660 root utmp
    rotate 1
}
EOF

# Validate
if ! logrotate -d /etc/logrotate.conf >/dev/null 2>&1; then
  echo "ERROR: Config validation failed!"
  logrotate -d /etc/logrotate.conf
  exit 1
fi

echo "Logrotate configured: ${ROTATE_DAYS} days retention"
echo "To test: logrotate -f /etc/logrotate.conf"
```

> This configuration sets a daily rotation schedule, compresses older logs, and retains a predictable number of days. It also preserves the standard wtmp/btmp rotation rules, which are traditionally handled monthly.

Test run:

```
logrotate -f /etc/logrotate.conf
```

> The `-f` flag forces rotation immediately. This is safe for testing and ensures your configuration behaves as expected.

---

### VII. Miscellaneous Operational Best Practices

This section captures additional Proxmox VE operational considerations that do not fit cleanly into other categories.

Topics include:
- Removal of downloading extra APT languages
- Post-install cleanup and validation tasks

#### 1. Skip downloading extra APT languages
This is a tweak that speeds up updates and saves a little disk. On servers, you usually do not need translation indexes for many different languages.
```
echo 'Acquire::Languages "none";' | tee /etc/apt/apt.conf.d/99-disable-translations
```
**Verify it applied:**
```
cat /etc/apt/apt.conf.d/99-disable-translations
```

---

### Closing Guidance

This guide is intended to provide clear, practical defaults that work well for most environments. Proxmox VE is flexible by design, and not all workloads have identical requirements. Administrators should treat these recommendations as a strong baseline and adjust only when there is a clear operational or technical justification.

---

## Resources

[Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/scripts)  
[12 Proxmox Host Tweaks Worth Doing This Weekend](https://www.virtualizationhowto.com/2025/12/12-proxmox-host-tweaks-worth-doing-this-weekend/)  
[PVE Post Install Script](https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install&category=Proxmox+%26+Virtualization)  
[User Management](https://pve.proxmox.com/wiki/User_Management#pveum_permission_management)  
[PVE Admin Guide - User Management](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#user_mgmt)  
[PVE Admin Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)