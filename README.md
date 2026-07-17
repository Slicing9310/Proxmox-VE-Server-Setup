# Proxmox-VE-Server-Setup

Welcome to my amateur Proxmox VE repository! This project serves as a log and storytelling space for my server setup, configuration, hosted services, and overall progress as I learn more about virtualization and homelabbing.

I am just a John Doe experimenting at home with spare parts I had lying around combined with a few components I bought. In essence, this project reflects my hunger for knowledge and a simple willingness to try new things.

### Network Isolation
My network is already separated into isolated VLANs. This Proxmox server is placed squarely on the Home Lab VLAN, and I would strongly advise anyone else to do the same—if only for maximum spousal approval if anything turns sour! 

### Design Philosophy
I watched hours of YouTube tutorials trying to figure out the perfect arrangement. Ultimately, advice from @WunderTech struck a chord with me: "...keep it as easy to maintain as possible". Because of that, I won't be installing anything overly complex or flashy. The primary purpose of this server is simply to give me hands-on insight into DNS servers, NAS configurations, ZFS, Windows Server, Linux Servers and FreeBSD.

---

## 🖥️ Hardware Specifications
*   **Host Machine:** Custom build
*   **CPU:** Intel® Core™ i7-12700 Processor (12 cores / 20 threads) 
*   **RAM:** 64GB DDR4 *(Note: Unfortunately running at a reduced 2133 MHz speed because one of the RAM sticks is damaged)*
*   **Storage:** 
    *   500GB NVMe SSD (Boot/VM OS storage)
    *   4TB NVMe SSD (Primary VM/LXC storage)
    *   2x 8TB 3.5" HDDs & 2x 4TB 3.5" HDDs (All consumer-grade drives, a few years old)
*   **GPU:** PNY NVIDIA T1000
*   **Network:** 10Gb SFP+ NIC connected via a DAC cable to a 10Gb Managed switch (equipped with SFP+ slots and additional 10Gb RJ45 ports)

---

## 🌐 Current Network & Architecture
My home features a Gigabit network running a gateway and a cloud management console, complete with IPS/IDS security. The network is segmented into 5 separate VLANs:
1.  Management Network
2.  Trusted Network
3.  Guest Network
4.  IoT Network
5.  Home Lab Network 

> 💡 *Note: IP address allocation is a highly personal choice, so feel free to use whatever subnet you prefer. The configurations below are fictional examples.*

*   **Subnet:** `172.188.200.0/24`
*   **Proxmox Host IP:** `172.188.200.10`

---

## 📦 Hosted Services (VMs & LXCs)
Following common advice found across tech YouTube, I map my VM/LXC IDs to match the host portion of their allocated static IP addresses.

| ID | Type | OS | Service Name | Purpose / Source |
| :--- | :--- | :--- | :--- | :--- |
| 100 | VM | Ubuntu Server | VM Template | Preconfigured base template via @LearnLinuxTV |
| 150 | LXC | Ubuntu Server | LXC Template | Preconfigured base container template via @Learn Linux TV (highly adviseable to all Linux fans) | 
| 200 | VM | Windows Server 2025| DC & AD | Domain Controller & Active Directory lab setup |
| 255 | VM | TrueNAS Scale | NAS Host | ZFS Mirrors, Samba/NFS shares, and iSCSI targets via @Tech-The Lazy Automator |

---

1. Day

## 🛠️ Base System Installation & Tweaks

### 1. Repository Configuration
Right after installation, the first step is updating the system repositories. I highly recommend using the automated helper script provided by the Proxmox VE Community Scripts site: https://community-scripts.org/

This script handles a few annoying post-install chores:
*   Deactivates the Enterprise repositories.
*   Enables the No-Subscription repository.
*   Disables the subscription "nag" pop-up window.
*   Disables High Availability (HA) services. Since I don't need HA and don't plan to build a cluster, keeping the setup minimal makes sense. 

*A quick system reboot is advised after completing the update process.*

### 2. Storage Preparation
The next step is configuring storage for actual VMs and containers:
*   I provisioned the **4TB NVMe SSD** as a single-disk ZFS pool.
*   Before creating pools on the remaining storage, it is best practice to double-check that the drives are completely cleared. If not, use the Proxmox GUI to wipe them.
*   Following a guide from @TechHutTV's home lab documentation, I removed the default `local-lvm` thin partition on the main 500GB boot NVMe. This allowed me to reclaim that space so the entire boot disk could be utilized efficiently.

### 3. Deploying TrueNAS & Troubleshooting Drive Visibility
My first major deployment was TrueNAS, utilizing the setup commands featured by @Tech-TheLazy Automator on YouTube. I followed the deployment steps closely while injecting my own preferences for ACLs and dataset management.

⚠️ **The First Major Roadblock:** 
During the initial setup, the underlying physical passthrough drives were not showing up properly inside the TrueNAS VM. The standard drive-mapping commands weren't passing enough unique data for TrueNAS to safely claim them, likely due to a lack of unique UUID mapping.

*   **The Original Command Structure:**
    ```bash
    qm set 255 -scsi1 /dev/disk/by-id/ata-Drive_Name
    qm set 255 -scsi2 /dev/disk/by-id/ata-Drive_Name
    ```
*   **My Solution:** 
    By appending an explicit, custom serial number string to the end of the argument, Proxmox forced the VM to recognize the virtualized hardware identities uniquely.
    ```bash
    qm set 255 -scsi1 /dev/disk/by-id/ata-Drive_Name,serial=xycydej
    qm set 255 -scsi2 /dev/disk/by-id/ata-Drive_Name,serial=xyvxnjd
    ```
### 4. Creating Shares & Data Pools
With the disk mapping resolved, I created a stable **8TB ZFS Mirror pool** using the two drives. From there, I configured my SMB, NFS, and iSCSI shares to distribute the storage space. 

> 💡 *Note: Just like IP configurations, storage space allocation, datasets, and ZVOLs are entirely subject to personal preference. Configure your capacities based on your specific lab requirements.*

---

## 🔮 Upcoming Milestones & Next Steps
- [ ] **Windows Server 2025 Setup:** My next guide will walk through how I deployed Windows Server 2025, promoted it to a Domain Controller (DC), and configured Active Directory (AD).
- [ ] **VM & LXC Templates:** After Windows Server, I will document how I built automated templates for Ubuntu Server and Alpine Linux based on @LearnLinuxTV's optimization strategies to spin up new environments in seconds.
