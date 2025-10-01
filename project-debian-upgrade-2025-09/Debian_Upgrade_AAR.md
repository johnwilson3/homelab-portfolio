# After-Action Report: Debian 12 to 13 Upgrade

**Project:** In-Place Distribution Upgrade of a Critical Proxmox VM

**Date:** September 24, 2025

**Owner:** John Wilson

### **1. Executive Summary**

This report documents the successful in-place upgrade of a critical virtual machine (`VMID 101`) from Debian 12 (Bookworm) to Debian 13 (Trixie). The project achieved its primary objective of enhancing system security and functionality with **zero data loss and a near-zero-downtime execution**, ensuring no impact on dependent services. A post-upgrade SSH connectivity issue was identified, a root cause was determined to be a configuration drift, and a permanent prevention strategy has been documented to standardize SSH configurations on all future VM deployments.

### **2. Objective**

The goal of this project was to perform a secure and stable in-place distribution upgrade of the primary Debian VM, which hosts critical containerized services. This upgrade brings the system up to date with the latest security patches, kernel enhancements, and software versions, ensuring the long-term stability and security of the home lab infrastructure.

### **3. Risk Mitigation & Rollback Plan**

A **redundant backup and recovery plan was implemented** prior to any system modifications to ensure multiple, independent recovery paths.

|   |   |   |
|---|---|---|
|**Layer**|**Strategy**|**Rationale & Verification**|
|**Layer 1: Full System**|**Full VM Snapshot (Proxmox/ZFS)**|A complete, point-in-time image of the VM was created. **A test restore was successfully performed** to a new VMID to validate the integrity of the backup.|
|**Layer 2: Guest Data**|**Full VM Backup (`vzdump`) to NAS**|Full, compressed `.vma.zst` backups of the VM were stored on a separate, resilient ZFS storage pool (TrueNAS), providing a robust, independent recovery asset.|
|**Layer 3: Config Files**|**Manual File Backup**|Key configuration files (`/etc/`, Docker `compose.yaml` files) were manually backed up off-host to enable a bare-metal rebuild if necessary.|

### **4.** Execution **Log Summary**

The upgrade was performed following the official Debian two-phase upgrade methodology. Key phases are summarized below with accurate timestamps from the operational log. (A detailed, time-stamped log with full command outputs is available for audit.)

1. **Pre-Upgrade Verification (Completed by 1:43 PM):** System state was verified, backups were confirmed as complete (10:52 AM), and a dry run was performed to assess the scope of the change (`1338 upgraded, 396 newly installed, 142 to remove`).
    
2. **Phase** 1 - **Minimal Upgrade (1:45 PM - 1:50 PM):** `sources.list` was updated to `trixie`. `sudo` apt `update` and `sudo apt upgrade --without-new-pkgs` were run to apply the minimal base system upgrade.
    
3. **Phase** 2 **- Full Distribution Upgrade (1:53 PM - 2:25 PM):** The `sudo apt full-upgrade` command was executed to handle all package dependencies and complete the transition to Debian 13.
    
4. **System Reboot & Cleanup (2:30 PM - 3:01 PM):** The system was rebooted into the new kernel. Post-reboot cleanup (`sudo apt autoremove`), re-enabling of the Docker repository, and updating of Docker packages were completed.
    

### **5. Post-Upgrade Verification**

Upon successful reboot, a full service validation checklist was performed to confirm system health and functionality.

|   |   |   |
|---|---|---|
|**Service**|**Status**|**Notes**|
|**Kernel Version**|✅ **PASS**|System correctly booted into Kernel `6.12.48+deb13-amd64`.|
|**Network Connectivity**|✅ **PASS**|VM maintained its static IP and had full network access.|
|**Docker Engine**|✅ **PASS**|Docker daemon started successfully.|
|**All Container Services**|✅ **PASS**|All 40+ containerized applications were started and confirmed to be fully functional.|
|**SSH Access**|❌ **FAIL (Initial)**|Initial SSH connection failed. See Post-Mortem (Remediated by 2:45 PM).|

### **6. Post-Mortem & Lessons Learned**

#### **Incident: Loss of SSH Access After Reboot**

- **Root Cause Analysis:** Upon investigation, it was determined that the `sshd_config` file on the VM had been previously modified but did not conform to the new, more secure defaults of the Debian 13 OpenSSH package (specifically regarding deprecated key exchange algorithms). The package upgrade process, correctly, did not overwrite the modified local configuration, resulting in a service failure.
    
- **Remediation:** The `sshd_config` was manually updated to align with the new package defaults, and the SSH service was successfully restarted.
    

#### **Actionable Prevention Strategy**

- **Finding:** The incident revealed a lack of configuration standardization across the lab's VMs.
    
- **Action Plan:** A baseline `sshd_config` template will be created and stored in a central Git repository. All future VM deployments will be provisioned using this standard template to prevent configuration drift. An audit of existing VMs will be conducted to align them with this new standard.
    

### **7. Stakeholder Impact & System Resilience**

The primary stakeholder for this environment is my wife, a critical remote worker. The upgrade was performed during her standard business hours, serving as a real-world test of the network's resilience.

Due to the network's architecture—specifically the strict isolation of her workstation on a dedicated VLAN with independent DNS—there was **zero impact** on her connectivity, application performance, or overall productivity throughout the entire upgrade process. The robust 10Gbe network backhaul also ensured that the download of update packages had no measurable effect on network latency.

This successful outcome serves as a practical validation of the network segmentation strategy, proving its effectiveness in ensuring business continuity for critical users during major infrastructure maintenance.
