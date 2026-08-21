# Proxmox VE (PVE) Initial Setup and Maintenance Guide

This is how I set up and maintain my Proxmox VE Server.

---

## 1. Post-Install Cleanup (Getting Rid of the Nag)

I start off by using a few pointers from *The Tech Hut* on YouTube. First up, running the post-install script from [community-scripts.org](https://community-scripts.org/scripts?q=nag&preview=post-pve-install).

This updates the repos and gets rid of the annoying subscription nag notification that pops up every single time you log in to remind you that you aren't paying for the enterprise edition.

Run this directly in your Proxmox VE shell:

```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/post-pve-install.sh)"

```

---

## 2. Reclaiming Space on the Boot Drive

I use my primary drive strictly for OS images and keep a separate NVMe drive for my VMs and LXCs. Proxmox advises using LVM partitioning upon installation, and while I don't plan on manually partitioning the boot drive, I *do* want to take advantage of the full NVMe storage space instead of letting Proxmox hoard half of it for an unused data partition.

Before deleting anything, double-check how your drive is partitioned in the Proxmox GUI or terminal.

Here are the commands to wipe the spare partition and merge it into the root OS:

```bash
# Remove the spare LVM data partition
lvremove /dev/pve/data

# Allocate 100% of the newly freed space to root
lvresize -l +100%FREE /dev/pve/root

# Resize the filesystem so the GUI actually shows the extra space
resize2fs /dev/mapper/pve-root

```

> **A Quick Disclaimer:** Proxmox separates the boot drive from VM storage for a very good reason—it’s a safety precaution. I do it this way because I have a secondary NVMe dedicated entirely to VMs and LXCs. But take a second to think about your setup: if you only have **one** single drive for your boot OS, VMs, and LXCs, **leave it as is**.

---

## 3. Enable IOMMU (PCIe Passthrough)

### What is IOMMU?

IOMMU (Input-Output Memory Management Unit) is the hardware component on modern motherboards that sits between system memory (RAM) and peripheral devices (GPUs, NICs, storage controllers).

While a standard MMU translates virtual memory addresses to physical RAM for the CPU, the IOMMU does that exact same memory translation and isolation layer for I/O devices. It is the foundational hardware requirement for secure virtualization, direct device passthrough, and keeping rogue PCIe devices from breaking your hypervisor.

### Step 1: Edit GRUB

To turn it on, we need to modify the default GRUB configuration file:

```bash
nano /etc/default/grub

```

Look for the line that says `GRUB_CMDLINE_LINUX_DEFAULT="quiet"`. Add `intel_iommu=on` or `amd_iommu=on` depending on your CPU brand:

```bash
# For Intel CPUs:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"

# For AMD CPUs:
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on"

```

* `Ctrl+O` then `Enter` to save, and `Ctrl+X` to exit.

*(Note: If you installed Proxmox on ZFS using `systemd-boot`, edit `/etc/kernel/cmdline` instead).*

### Step 2: Apply Changes & Reboot

Now update GRUB and reboot the machine so the kernel updates initramfs:

```bash
update-grub
reboot

```

### Step 3: Check Your Work

I usually hate double-checking my progress, but at this stage, I strongly advise it. Run these commands to verify that IOMMU is active:

```bash
dmesg | grep -e DMAR -e IOMMU
dmesg | grep 'remapping'

```

Look for these key messages:

* `DMAR: IOMMU enabled` → Kernel detected your Intel VT-d / AMD-Vi configuration.
* `DMAR-IR: Enabled IRQ remapping in x2apic mode` → Interrupt remapping is active (crucial for routing GPU interrupts directly to a VM).
* `DMAR: Intel(R) Virtualization Technology for Directed I/O` → It's loaded and active.

If your terminal output doesn't match this... well, I am sorry to say, but you need to start digging. Pipe `dmesg` into `less` and look for errors. Proxmox VE is built on Debian, so the system is usually pretty generous with error logs. That’s what I love about Linux: if you're willing to invest a little concentration, the debugging tools will make the time investment worth it—even if you're a Linux newbie like myself.

*Alright, enough fanboy nonsense. Moving on.*

---

## 4. Automated Updates with `unattended-upgrades`

Linux gives you the power of unattended upgrades—meaning the system will upgrade itself without you having to lift a finger.

Why bother? Because manually running update commands every single day gets old fast. Plus, an updated system is a less vulnerable system.

> **Important Caveat:** If your server reboots after an update (like a kernel upgrade), any VMs and LXCs **not** explicitly configured to "Start at Boot" will remain offline. Keep that in mind.

First, verify that your server can actually handle a manual reboot without hanging:

```bash
reboot

```

Once you've confirmed your system actually comes back alive, install `unattended-upgrades`:

```bash
apt update && apt install -y unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades

```

*(The `--priority=low` flag forces the setup wizard to display all the hidden, advanced prompts instead of quietly choosing defaults for you).*

Verify the service is actually running:

```bash
systemctl status unattended-upgrades

```

If it’s enabled and running, all is good in Proxmox Land. If not? Hard luck! Just kidding—use `systemctl enable --now unattended-upgrades` to force it on.

### Tweaking the Config Files

Next, check `/etc/apt/apt.conf.d/20auto-upgrades` to make sure auto-upgrades are enabled at the APT level:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades

```

Both entries should be set to `1` (binary shorthand: `0` = off, `1` = on). If you see a `0`, change it using your text editor of choice (`nano` or `vim`—if you use `vim`, I bow to you, but you probably aren't reading my GitHub anyway).

Now let's modify `/etc/apt/apt.conf.d/50unattended-upgrades`, which holds all the granular settings. Before touching an important config file, make a backup:

```bash
cp /etc/apt/apt.conf.d/50unattended-upgrades /etc/apt/apt.conf.d/80unattended-upgrades
nano /etc/apt/apt.conf.d/80unattended-upgrades

```

*(Naming it `80-` ensures our preferences override the original file and won't get wiped during package upgrades).*

I strongly advise reading through the file just to see the kaleidoscopic array of options Linux gives you. (Sidenote: while some config files use `#` for comments, this one uses `//`).

Here are the specific lines you'll want to uncomment and tweak:

```cpp
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";

```

*Note: If you run `unattended-upgrades` inside your guest VMs too, set their reboot times to trigger **after** the host server's reboot time.*

### Controlling the Execution Window via Systemd

When do these upgrades actually trigger? By default, whenever systemd feels like it. To set a predictable schedule, edit the systemd timer:

```bash
systemctl edit apt-daily-upgrade.timer

```

Add your preferred execution time (e.g., triggering updates shortly before your planned 02:00 reboot window):

```ini
[Timer]
OnCalendar=*-*-* 01:20:00
RandomizedDelaySec=600

```

Finally, reload and check your work to make sure everything took effect:

```bash
# Reload systemd configs
systemctl daemon-reload

# Restart the service
systemctl restart unattended-upgrades

# Check timer status (look at the 'Trigger' line for the countdown)
systemctl status apt-daily-upgrade.timer

```

---

That’s enough for today. My next step will be setting up non-root admin users, because logging directly into `root` for daily homelab tasks is terrible practice—and building good habits now will pay off later in enterprise environments.

Hello World !!!

### 5. Managing Users and Granular Permissions

As promised, now we will tackle user management. There is allegedly a best practice prescribing that we manage Linux machines using a dedicated user—**NOT ROOT**. It makes sense: if your credentials get leaked, at least you can delete that user and start over without losing the entire hypervisor.

In Proxmox, navigate to **Datacenter**, scroll down to **Permissions**, and click on the **Users** tab. If this is a fresh setup, you will only see `root@pam`.

An important distinction to understand here is the difference between realm types:

* **PAM (`@pam`):** These correspond directly to underlying Linux system users (used for SSH access to the base Debian OS). You can create a matching system user in the shell by running:
```bash
adduser xyz

```


Fill out the prompts and **do not forget** to add this user to the `sudo` group (`usermod -aG sudo xyz`), or you will lock yourself out of administrative SSH access.
* **PVE (`@pve`):** These are Proxmox-native users created purely inside the GUI for managing hypervisor resources.

Once created, you’ll notice the user has no real access yet because they haven’t been assigned permissions or a group. You *could* assign permissions directly to a single user, but as your homelab grows, this gets messy fast. Best practice is to assign permissions to a **Group**.

1. Go to the **Groups** tab and click **Create**.
2. Give it a name and a comment (helpful if the naming scheme is abstract).
3. Head over to **Permissions** and assign the appropriate role to the group.

Personally, I assign my daily admin user slightly lower permissions than root. A good admin doesn't need all-powerful access for basic daily tasks—and shouldn't become a single point of failure. Imagine being on vacation, having the time of your life, and suddenly having to remotely debug a critical permission mess you made because you were logged in as superuser.

Add your newly created user to the group, log out of root, and log back in as your daily driver to test your access.

---

### 6. Storage & Hyperconverged NAS: TrueNAS SCALE VM

I don't have infinite financial resources for my backup and home services, so I use this PVE server to consolidate as much as possible. My favorite approach starts with setting up a **TrueNAS SCALE** VM for SMB/NFS/iSCSI storage and backups.

I’ve heard all the lectures about why you shouldn't run TrueNAS in a VM. You guys are technically right, but since this is my server, I do what I want with it!

Why TrueNAS SCALE? One word: **ZFS**.

Why not just use ZFS directly on Proxmox? I take ZFS wherever I can get it. I use Proxmox's built-in ZFS for VM/LXC storage performance, and TrueNAS's ZFS implementation to manage media and phone photo backups.

#### Passthrough Disks by ID

My setup uses a primary NVMe drive for core VMs/LXCs, plus two mechanical HDDs in a ZFS mirror for extra virtual storage. For TrueNAS, I have two *additional* dedicated mechanical drives that need to be passed through directly using their unique disk IDs; otherwise, TrueNAS won't recognize them properly.

1. Download the TrueNAS SCALE ISO into Proxmox and create the VM.
2. Open the Proxmox terminal to identify your physical drives:
```bash
lsblk

```


3. List the unique serial-based disk paths:
```bash
ls /dev/disk/by-id/

```


4. Cross-reference the serial numbers with the **Disks** tab under your Proxmox node in the GUI to ensure you have the exact right drives.
5. Pass the raw drives directly to your TrueNAS VM (assuming VM ID `100`):
```bash
qm set 100 -scsi1 /dev/disk/by-id/ata-WD7000CM006-2ZSFD2_XS345FZAC,serial=XS345FZAC
qm set 100 -scsi2 /dev/disk/by-id/ata-WD5000FM008-5ZSDF4_FST5HGSTZ,serial=FST5HGSTZ

```



*Breakdown:* `qm set [VM_ID]` tells Proxmox which VM to modify, `-scsiX` specifies the virtual controller slot, and passing the `/dev/disk/by-id/...` path routes the physical disk. In my setup, explicitly defining the `,serial=` flag at the end was mandatory—Proxmox refused to surface the drives inside TrueNAS properly without it.

Boot into TrueNAS, hit the Web UI, and your raw drives will be ready for pool creation. If I ever build a second dedicated hardware box, I'll run TrueNAS bare-metal, but inside PVE it works amazingly well.

---

### 7. Media Streaming: Dedicated Jellyfin VM

A lot of online guides suggest running Jellyfin inside an LXC container and mounting media via an SMB share. In my testing, that introduced unnecessary instability. I read through the official documentation and decided to give Jellyfin its own dedicated Ubuntu VM instead.

#### GPU Hardware Transcoding Reality Check

If you want to stream 4K or HDR content without melting your CPU, you **must** pass through a dedicated GPU for hardware transcoding.

Online opinions on graphics cards vary wildly, but here is my actual experience testing two budget GPUs:

* **PNY NVIDIA T1000 (4GB):** Performed marvelously at 1080p, but struggled hard with heavy 4K HDR streams.
* **Sparkle Intel Arc A310 OMNI:** Outperformed the NVIDIA card without breaking a sweat. Intel QuickSync on the Arc lineup is incredible for media servers.

#### PCI Passthrough Setup

To dedicate the Intel Arc GPU to the Jellyfin VM:

1. Go to the VM's **Hardware** tab in Proxmox.
2. Add a **PCI Device**, select the GPU as a **Raw Device**, enable **All Functions**, and leave *Primary GPU* **unchecked**.
3. Do this *before* installing Ubuntu so the installer automatically picks up the third-party hardware drivers during setup.

#### Setting Up Storage & Filesystems

1. Install Ubuntu Server, update it (`sudo apt update && sudo apt upgrade -y`), and follow Jellyfin's official repository installation script.
2. Add your local Linux user to the GPU rendering group:
```bash
sudo usermod -aG render $USER

```


3. To give Jellyfin access to extra storage from the host's ZFS mirror pool, attach a new virtual disk via the Proxmox GUI **Hardware** tab.
4. Partition and format the new disk inside the VM:
```bash
# Create a GUID Partition Table (GPT) for drives > 2TiB
sudo fdisk /dev/sdX
# Type 'g' (new GPT), 'p' (print/verify), 'w' (write and exit)

# Format as ext4
sudo mkfs.ext4 /dev/sdX1

```


5. Create a target directory and mount it. Avoid mounting random folders straight to the root directory `/`—keep it clean by mounting under `/home` or `/mnt`:
```bash
sudo mkdir -p /home/media
sudo mount /dev/sdX1 /home/media

```


6. Make the mount permanent by getting the disk UUID (`sudo blkid`) and adding it to `/etc/fstab`, then reload systemd:
```bash
sudo mount -a
sudo systemctl daemon-reload

```



#### Moving Data with `rsync`

To transfer existing media libraries over the network, **pushing** the data from the source machine to the destination is best practice. I use `rsync` with the `-P` flag so I can track progress:

```bash
rsync -avzP /source/directory/ user@192.168.1.100:/home/media/

```

Once the transfer completes, log into the Jellyfin Web UI, point your media libraries to `/home/media`, and enable Intel QuickSync (QSV) in the Transcoding settings.

---

### What's Next?

That wraps up the storage and media side of my server. My next project is spinning up a **Windows Server 2025** VM to attach a few test PCs and experiment with Active Directory (AD) and Domain Controller (DC) configurations. Stay tuned for Part 3!
That wraps up the storage and media side of my server. My next project is spinning up a Windows Server 2025 VM to attach a few test PCs and experiment with Active Directory (AD) and Domain Controller (DC) configurations. Stay tuned for Part 3!
