# After-Action Report: Proxmox Backup Server (PBS) Deployment & Architectural Hardening

Project: Strategic Migration from vzdump to a Dedicated Proxmox Backup Server Instance

Date: September 29, 2025

Owner: John Wilson

### 1. Executive Summary

This report documents the successful deployment and stabilization of a centralized Proxmox Backup Server (PBS) instance. The project was initiated to proactively upgrade the lab's data protection strategy from a basic vzdump model to a more efficient and resilient PBS architecture.

During initial testing, a critical architectural flaw in the legacy vzdump process was discovered: it temporarily stages backup data on the host's local LVM storage before moving it to the final destination. This caused a cascading service failure on the secondary Proxmox VE node (PVE-Node2), validating the strategic necessity of deploying a dedicated PBS solution. The project culminated in a fully validated, end-to-end data protection system that now serves as the foundation for a future enterprise-grade 3-2-1 backup strategy.

### 2. Strategic Objective & Scope

The primary objective was to proactively engineer a modern, dedicated backup solution to replace an architecturally flawed legacy method. The key goals were to leverage PBS's efficiency (deduplication, incrementals) and to establish a resilient data protection baseline before undertaking major infrastructure changes.

### 3. Incident Timeline & Architectural Validation

1. Project Initiation: The project was initiated to migrate from a legacy vzdump backup strategy to a more efficient, centralized PBS solution.
    
2. Test Case: An initial test was conducted by performing a vzdump backup of a large VM (VM 201, 500GB) from PVE-Node1 to an external HDD attached to the secondary node, PVE-Node2.
    
3. Architectural Flaw Identified: The vzdump process began staging the 500GB backup file on the local-lvm of PVE-Node2 (375GB capacity), causing the LVM to saturate and triggering a cascading failure of all guest services on that node.
    
4. Root Cause Analysis: The test successfully proved that using a hypervisor's OS storage as a temporary staging area for large backups is an architectural anti-pattern and not a resilient or scalable solution. This validated the strategic decision to proceed with a dedicated PBS deployment using a network-attached datastore.
    
5. PBS Deployment: The project continued with the deployment of PBS into a Privileged LXC Container, configured to use a dedicated, network-attached TrueNAS NFS share.
    

### 4. Post-Deployment Incident: "OS Error 13"

During initial backup tests with the new, correct architecture, a persistent "OS Error 13 (Permission Denied)" failure was encountered when the PBS instance attempted to write backup chunks to the NFS share.

- Root Cause Analysis: A deep-dive investigation revealed a classic cross-platform UID/GID mismatch. The PBS service, running as the unprivileged backup user (UID 34) within its container, was denied write access by the underlying ZFS filesystem permissions on the TrueNAS dataset, which was owned by root.
    
- Remediation: The ownership of the TrueNAS dataset was recursively changed to backup:backup, aligning the filesystem permissions with the application-level user and resolving the conflict.
    

### 5. Actionable Prevention Strategy

- Finding: The incident revealed the critical importance of coordinating both network-level (NFS Maproot) and filesystem-level (ZFS dataset owner) permissions in a multi-system environment.
    
- Action Plan: A new Standard Operating Procedure (SOP) has been created for all future NFS share provisioning for containerized services. The SOP mandates that the dataset owner and group must match the UID/GID of the service user within the container.
    

### 6. Validation & Future State Architecture

Following the remediation, a full backup of a test VM was successfully completed, and a test restore was performed and validated, confirming the end-to-end integrity of the new data protection system.

The PBS instance is now fully operational, and the lab's backup architecture has been successfully upgraded from a fragile legacy model to a documented, resilient, and modern solution. This project has unblocked the path for the lab's future-state data protection architecture, which includes:

1. Phase 1 (Complete): Stabilize backups with a virtualized PBS instance.
    
2. Phase 2 (Next): Migrate PBS to a dedicated physical appliance (N100 Mini PC) to create a truly independent backup server.
    
3. Phase 3 (Future): Implement a full 3-2-1 backup strategy by configuring PBS sync jobs to the TrueNAS array (2nd copy) and a subsequent cloud backup job (3rd, off-site copy).
