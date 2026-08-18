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
