# Creating a Standalone KVM VM from a CloudStack Instance Snapshot

**Purpose:** Take a snapshot of a CloudStack instance's ROOT volume, convert it into a reusable template, then use that template to build a standalone libvirt/KVM VM directly on the hypervisor — outside CloudStack management.

**Environment:**
- CloudStack UI: accessible via management server
- Hypervisor host: `cloudstack-node1` (KVM)
- CloudStack secondary storage: `172.16.1.71:/export/secondary`
- Bridge: `cloudbr0`
- Result VM: `myvm-clone-01`

---

## Table of Contents

- [PART A — CloudStack: Snapshot the Instance Volume](#part-a--cloudstack-snapshot-the-instance-volume)
- [PART B — CloudStack: Create a Template from the Snapshot](#part-b--cloudstack-create-a-template-from-the-snapshot)
- [PART C — KVM Host: Build a Standalone VM from the Template](#part-c--kvm-host-build-a-standalone-vm-from-the-template)
- [How the VM Gets Its IP](#how-the-vm-gets-its-ip-network-flow)
- [Common Issues & Fixes](#common-issues--fixes)
- [Reference Summary](#reference-summary)
- [Useful virsh Commands](#useful-virsh-commands)

---

## PART A — CloudStack: Snapshot the Instance Volume

### Step A1 — Stop the VM

1. Log in to the CloudStack UI.
2. Go to **Compute → Instances**.
3. Select your VM.
4. Click **Stop**.
5. Wait until the VM state shows **Stopped**.

> Volume snapshots require the VM to be stopped in most CloudStack KVM setups. Snapshots of running VMs are disabled by default on KVM.

---

### Step A2 — Take a Snapshot of the ROOT Volume

1. Go to **Storage → Volumes** (or open the VM → **Volumes** tab).
2. Find the **ROOT** volume of your VM.
   - Must be the ROOT volume, not a data volume.
   - Volume state must be **Ready**.
3. Select the volume → click **Take Snapshot**.
4. Confirm.

**Wait** for status: `Creating` → **`BackedUp`**.

> If "Take Snapshot" is missing: confirm the VM is stopped, confirm secondary storage exists for the zone, then refresh the page.

---

## PART B — CloudStack: Create a Template from the Snapshot

### Step B1 — Open the Snapshots List

1. Go to **Storage → Snapshots**.
2. Locate the snapshot you just took.
   - Status must be **BackedUp**.
   - Must be the **ROOT** volume of the correct VM.

---

### Step B2 — Create the Template

1. Select the snapshot → click **Create Template**.
2. Fill in the form:

| Field | Value |
|---|---|
| Name | e.g., `cloudstack-snaptemplate` |
| Display Text | e.g., `Template from VM snapshot` |
| OS Type | **Must match the original VM's OS exactly** |
| Public | Unchecked |
| Featured | Unchecked |
| Password Enabled | **UNCHECKED** if deploying to an OVN network |
| Dynamically Scalable | Unchecked (optional) |
| HVM | Unchecked (KVM/VMware) / Checked (XenServer + Windows) |

3. Click **OK**.

> **Important — Password Enabled must be unchecked** if you plan to deploy to an OVN network. Otherwise CloudStack throws: *"Template is password enabled, but there is no support for UserData service in the default network."*

---

### Step B3 — Wait for the Template to Become Ready

1. Go to **Images → Templates**.
2. Find your template.
3. Wait for status: `Creating` → **`Ready`** (10–30 minutes typical).

**Note the template ID** (e.g., `256`) — you'll need it later.

--

# PART C — KVM Host: Build a Standalone VM from the Template

## Step C1 — Mount CloudStack Secondary Storage

### Prerequisites (on the KVM host)

```bash
sudo apt update
sudo apt install -y virtinst qemu-utils
```

### Check NFS Export

```bash
showmount -e 172.16.1.71
```

### Expected Output

```text
Export list for 172.16.1.71:
/export/secondary 172.16.1.0/24
```

### Create the Mount Point

```bash
sudo mkdir -p /mnt/sec
```

### Mount CloudStack Secondary Storage

```bash
sudo mount -t nfs 172.16.1.71:/export/secondary /mnt/sec
```

### Verify the Mount

```bash
mount | grep /mnt/sec
```

```bash
ls -la /mnt/sec
```

### Expected Folders

```text
template/
volumes/
snapshots/
```

---

## Step C2 — Locate the Template File

Find the QCOW2 template file:

```bash
sudo find /mnt/sec/template -iname "*.qcow2"
```

### Example Output

```text
/mnt/sec/template/tmpl/2/256/0b2b1064-7f65-4373-876c-c8f6cfa8a96f.qcow2
```

> `2` = Account ID
> `256` = Template ID

### Inspect the Template

```bash
sudo qemu-img info /mnt/sec/template/tmpl/2/256/0b2b1064-7f65-4373-876c-c8f6cfa8a96f.qcow2
```

```bash
sudo qemu-img check /mnt/sec/template/tmpl/2/256/0b2b1064-7f65-4373-876c-c8f6cfa8a96f.qcow2
```

### Expected

The output should show:

* `qcow2` format
* Virtual size
* `No errors were found`

---

## Step C3 — Copy the Template to Local Storage

Copy the template from NFS to the local KVM storage:

```bash
sudo cp /mnt/sec/template/tmpl/2/256/0b2b1064-7f65-4373-876c-c8f6cfa8a96f.qcow2 \
        /var/lib/libvirt/images/template.qcow2
```

### Verify the File Size

```bash
sudo stat -c '%s' /var/lib/libvirt/images/template.qcow2
```

```bash
sudo stat -c '%s' /mnt/sec/template/tmpl/2/256/0b2b1064-7f65-4373-876c-c8f6cfa8a96f.qcow2
```

### Verify the QCOW2 File

```bash
sudo qemu-img check /var/lib/libvirt/images/template.qcow2
```

Both `stat` values should match.

`qemu-img check` should report:

```text
No errors were found
```

> **Note:** Large copies over NFS can take several minutes. Do not interrupt `cp` while it is in `D` (uninterruptible I/O) state. This can be normal for NFS I/O.

---

## Step C4 — Create the VM's Writable Disk

Go to the libvirt image directory:

```bash
cd /var/lib/libvirt/images
```

Create a copy for the new VM:

```bash
sudo cp template.qcow2 myvm-clone-01.qcow2
```

Verify:

```bash
sudo ls -lh myvm-clone-01.qcow2
```

---

## Step C5 — Identify the Network Bridge

Check the available Linux bridges:

```bash
brctl show
```

### Example Output

```text
bridge name   bridge id             STP enabled   interfaces
cloudbr0      8000.c6ce01a177cc     no            enp130s0
```

Use the bridge connected to the physical NIC.

In this setup:

```text
cloudbr0
```

---

## Step C6 — Create the VM with virt-install

Create the VM using the existing QCOW2 disk:

```bash
sudo virt-install \
  --name myvm-clone-01 \
  --memory 2048 \
  --vcpus 2 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/myvm-clone-01.qcow2,format=qcow2,bus=virtio \
  --os-variant generic \
  --network bridge=cloudbr0,model=virtio \
  --graphics vnc,listen=0.0.0.0 \
  --console pty,target_type=serial \
  --import \
  --noautoconsole
```

### Important Options

| Option                      | Meaning                                           |
| --------------------------- | ------------------------------------------------- |
| `--import`                  | Boot an existing disk instead of installing an OS |
| `--disk path=...`           | Specifies the VM's QCOW2 disk                     |
| `--network bridge=cloudbr0` | Connects the VM to `cloudbr0`                     |
| `--graphics vnc`            | Enables VNC graphics                              |
| `--console pty`             | Enables serial console                            |
| `--noautoconsole`           | Does not automatically attach to the console      |

---

## Step C7 — Verify the VM

Check all VMs:

```bash
sudo virsh list --all
```

Check VM details:

```bash
sudo virsh dominfo myvm-clone-01
```

Check the VM IP:

```bash
sudo virsh domifaddr myvm-clone-01
```

The VM should show:

```text
State: running
```

> `domifaddr` may take approximately 30 seconds to display the IP.

### If No IP Is Displayed

Check the VM's network interface:

```bash
sudo virsh domiflist myvm-clone-01
```

Then check the ARP/neighbour table:

```bash
ip neigh | grep <mac>
```

---

## Step C8 — Access the VM

### Serial Console

```bash
sudo virsh console myvm-clone-01
```

Exit the serial console using:

```text
Ctrl+]
```

### VNC

Check the VNC display:

```bash
sudo virsh domdisplay myvm-clone-01
```

Example:

```text
vnc://127.0.0.1:5900
```

### Create an SSH Tunnel from the Laptop

```bash
ssh -L 5900:127.0.0.1:5900 appsquadz@cloudstack-node1
```

### SSH into the VM

```bash
ssh generic@<vm-ip>
```

> Use the original VM's credentials because the template preserves the original credentials.

---

## Step C9 — Post-Clone Cleanup

Run the following commands **inside the cloned VM**.

### 1. Set a New Hostname

```bash
sudo hostnamectl set-hostname myvm-clone-01
```

### 2. Generate a New Machine ID

```bash
sudo rm -f /etc/machine-id /var/lib/dbus/machine-id
sudo systemd-machine-id-setup
```

### 3. Regenerate SSH Host Keys

```bash
sudo rm -f /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo systemctl restart sshd
```

### 4. Reboot the VM

```bash
sudo reboot
```

> Also update `/etc/hosts` if it contains the old hostname.

---

## Step C10 — Post-Deploy Verification

### On the KVM Host

```bash
sudo virsh list --all
```

```bash
sudo virsh dominfo myvm-clone-01
```

```bash
sudo virsh domifaddr myvm-clone-01
```

```bash
sudo virsh domblklist myvm-clone-01
```

```bash
sudo virsh domiflist myvm-clone-01
```

### Inside the VM

```bash
ip a
```

```bash
ip route
```

```bash
ping -c 3 <gateway>
```

```bash
ping -c 3 8.8.8.8
```

```bash
ping -c 3 google.com
```

```bash
hostname
```

```bash
lsblk && df -h
```

```bash
nproc && free -h
```

```bash
systemctl --failed
```

```bash
timedatectl
```

---

## Optional — Enable VM Autostart

To automatically start the VM when the KVM host reboots:

```bash
sudo virsh autostart myvm-clone-01
```

---

## Optional — Create a Baseline Snapshot

Create a clean baseline snapshot:

```bash
sudo virsh snapshot-create-as myvm-clone-01 clean-baseline "Initial clone"
```

Verify:

```bash
sudo virsh snapshot-list myvm-clone-01
```

---

# How the VM Gets Its IP

```text
[ VM ens3: MAC 52:54:00:xx:xx:xx ]
              │
              │ virtio
              ▼
[ Host tap: vnetX ]
              │
              │ attached to
              ▼
[ Host bridge: cloudbr0 ]
              │
              │ bridged to physical NIC
              ▼
[ Physical NIC: enp130s0 ]
              │
              ▼
[ LAN Switch ]
              │
              ▼
[ DHCP Server ]
              │
              ▼
[ VM receives 172.16.4.x/21 ]
```

### Network Flow

1. `virt-install --network bridge=cloudbr0` attaches the VM to `cloudbr0`.
2. The VM sends a DHCP `DISCOVER`.
3. The DHCP request passes through the bridge.
4. The LAN DHCP server assigns an IP address.
5. `cloud-init` applies the IP configuration to `ens3`.

### Verify from the Host

```bash
sudo virsh domiflist myvm-clone-01
```

```bash
ip neigh | grep <vm-mac>
```

---

# Common Issues & Fixes

| Problem                                                | Cause                              | Fix                                               |
| ------------------------------------------------------ | ---------------------------------- | ------------------------------------------------- |
| "Take Snapshot" option missing                         | VM running or no secondary storage | Stop VM and verify secondary storage              |
| "Template is password enabled, but no UserData on OVN" | Password Enabled + OVN network     | Recreate template with Password Enabled unchecked |
| `mount.nfs: /mnt/sec does not exist`                   | Mount point missing                | `sudo mkdir -p /mnt/sec`                          |
| `mount.nfs: access denied`                             | Wrong export path                  | Use `/export/secondary`                           |
| `cp` hangs on NFS                                      | Slow NFS / hard mount              | Wait or use `qemu-img convert`                    |
| `Permission denied` on QCOW2                           | Wrong ownership                    | `sudo chown libvirt-qemu:kvm <file>`              |
| VM boots to GRUB rescue                                | Wrong disk bus                     | Change `bus=virtio` to `bus=sata`                 |
| No IP inside VM                                        | Bridge / DHCP issue                | Check `brctl show` and verify DHCP                |
| SSH host key warning                                   | Clone kept original keys           | Regenerate SSH host keys                          |
| VM stuck in `paused`                                   | Disk I/O error                     | Check `journalctl -u libvirtd`                    |
| Blank console                                          | No serial console configured       | Use VNC or add `console=ttyS0`                    |

---

# Reference Summary

| Item                 | Value                                         |
| -------------------- | --------------------------------------------- |
| Original VM          | CloudStack instance (ROOT volume snapshotted) |
| Snapshot location    | CloudStack secondary storage                  |
| Template ID          | e.g. `256`                                    |
| NFS server           | `172.16.1.71:/export/secondary`               |
| NFS mount            | `/mnt/sec`                                    |
| Template source file | `/mnt/sec/template/tmpl/2/256/<uuid>.qcow2`   |
| Local template       | `/var/lib/libvirt/images/template.qcow2`      |
| VM disk              | `/var/lib/libvirt/images/myvm-clone-01.qcow2` |
| VM name              | `myvm-clone-01`                               |
| Bridge               | `cloudbr0`                                    |
| IP assignment        | DHCP via bridged LAN                          |

---

# Useful virsh Commands

| Action                | Command                                    |
| --------------------- | ------------------------------------------ |
| List VMs              | `virsh list --all`                         |
| Show details          | `virsh dominfo <vm>`                       |
| Start                 | `virsh start <vm>`                         |
| Graceful shutdown     | `virsh shutdown <vm>`                      |
| Force off             | `virsh destroy <vm>`                       |
| Reboot                | `virsh reboot <vm>`                        |
| Serial console        | `virsh console <vm>`                       |
| VNC address           | `virsh domdisplay <vm>`                    |
| List disks            | `virsh domblklist <vm>`                    |
| List NICs             | `virsh domiflist <vm>`                     |
| Delete VM (keep disk) | `virsh undefine <vm>`                      |
| Delete VM + disk      | `virsh undefine <vm> --remove-all-storage` |
| Enable autostart      | `virsh autostart <vm>`                     |

