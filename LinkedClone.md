# Method 1: Linked Clone from Golden Image — Full Playbook

Create unlimited new Ubuntu VMs from a single golden image in ~2 seconds each, using qcow2 linked clones.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [One-Time Setup](#-one-time-setup-do-these-only-once)
  - [A. Prepare the Base VM](#a-prepare-the-base-vm-inside-iphone)
  - [B. Create Golden Images](#b-create-golden-images-host)
  - [C. Install the Clone Script](#c-install-the-clone-script-host)
- [Recurring Workflow](#-recurring-workflow--create-a-new-linked-clone)
- [Management Commands](#-management-commands)
- [Golden Rules](#️-golden-rules)

---

## Prerequisites

| Item | Required |
|---|---|
| Host OS | RHEL-based (uses `dnf`, `virsh`, LVM) |
| Host tools | `qemu-img`, `virsh`, `lvcreate`, `lvremove` |
| Source VM | Raw LVM disks (uses `/dev/vg_vm/iphone`, `/dev/vg_vm/iphone01`) |
| Free space | At least ~15 GB in `/var/lib/libvirt/images/` |
| Network bridge | `br1` (adjust if different) |

---

## 🎯 One-Time Setup (do these only once)

### A. Prepare the Base VM (inside `iphone`)

#### Boot the VM (run from HOST)

```bash
sudo virsh start iphone
sudo virsh console iphone
```

#### Confirm you're inside the VM

```bash
hostname
```

Must say `iphone` or `golden-template-iphone` — **not** `cloudstack`.

> **Tip:** To exit the VM console at any point, press `Ctrl + ]`.

#### Install whatever you need baked into the golden image

```bash
# Example — install Docker
sudo apt update && sudo apt install -y docker.io
```

Install any other tools, configs, or packages you want every clone to inherit.

#### Generalize the system so clones don't conflict

```bash
sudo hostnamectl set-hostname golden-template
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo cloud-init clean --logs --seed 2>/dev/null || true
history -c
```

#### Power off the VM

```bash
sudo shutdown -h now
```

Exit the console with `Ctrl + ]`.

---

### B. Create Golden Images (HOST)

Back on the **host** prompt (`[appsquadz@cloudstack ~]$`), verify the VM is off:

```bash
sudo virsh domstate iphone
```

Must say **`shut off`**.

#### Create LVM snapshots of the raw disks

```bash
sudo lvcreate -L 5G -s -n iphone_golden_snap   /dev/vg_vm/iphone
sudo lvcreate -L 3G -s -n iphone01_golden_snap /dev/vg_vm/iphone01
```

#### Convert them to qcow2 golden images

```bash
sudo qemu-img convert -p -f raw -O qcow2 \
  /dev/vg_vm/iphone_golden_snap \
  /var/lib/libvirt/images/golden-iphone.qcow2
```

```bash
sudo qemu-img convert -p -f raw -O qcow2 \
  /dev/vg_vm/iphone01_golden_snap \
  /var/lib/libvirt/images/golden-iphone01.qcow2
```

#### Lock them read-only

```bash
sudo chmod 444 /var/lib/libvirt/images/golden-iphone.qcow2
sudo chmod 444 /var/lib/libvirt/images/golden-iphone01.qcow2
```

#### Verify

```bash
sudo ls -lh /var/lib/libvirt/images/ | grep golden
sudo qemu-img info /var/lib/libvirt/images/golden-iphone.qcow2
```

Both files should show mode `-r--r--r--`.

#### Remove the LVM snapshots (space reclaim)

```bash
sudo lvremove /dev/vg_vm/iphone_golden_snap
sudo lvremove /dev/vg_vm/iphone01_golden_snap
```

---

### C. Install the Clone Script (HOST)

Create `/usr/local/bin/make-clone.sh`:

```bash
sudo tee /usr/local/bin/make-clone.sh > /dev/null <<'SCRIPT'
#!/bin/bash
# Usage: make-clone.sh <clone-name>
set -e

NAME="$1"
if [ -z "$NAME" ]; then
  echo "Usage: $0 <clone-name>"
  exit 1
fi

GOLDEN1="/var/lib/libvirt/images/golden-iphone.qcow2"
GOLDEN2="/var/lib/libvirt/images/golden-iphone01.qcow2"
CLONE1="/var/lib/libvirt/images/${NAME}.qcow2"
CLONE2="/var/lib/libvirt/images/${NAME}01.qcow2"

qemu-img create -f qcow2 -F qcow2 -b "$GOLDEN1" "$CLONE1"
qemu-img create -f qcow2 -F qcow2 -b "$GOLDEN2" "$CLONE2"

cat > /tmp/${NAME}.xml <<EOF
<domain type='kvm'>
  <name>${NAME}</name>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>4</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-rhel9.8.0'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features><acpi/><apic/></features>
  <cpu mode='host-passthrough' check='none' migratable='on'/>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/>
      <source file='${CLONE1}'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='${CLONE2}'/>
      <target dev='vdb' bus='virtio'/>
    </disk>
    <controller type='usb' index='0' model='qemu-xhci' ports='15'/>
    <controller type='pci' index='0' model='pcie-root'/>
    <controller type='pci' index='1' model='pcie-root-port'/>
    <controller type='pci' index='2' model='pcie-root-port'/>
    <controller type='sata' index='0'/>
    <controller type='virtio-serial' index='0'/>
    <interface type='bridge'>
      <source bridge='br1'/>
      <model type='virtio'/>
    </interface>
    <serial type='pty'><target type='isa-serial' port='0'><model name='isa-serial'/></target></serial>
    <console type='pty'><target type='serial' port='0'/></console>
    <channel type='unix'><target type='virtio' name='org.qemu.guest_agent.0'/></channel>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <audio id='1' type='none'/>
    <memballoon model='virtio'/>
    <rng model='virtio'><backend model='random'>/dev/urandom</backend></rng>
  </devices>
</domain>
EOF

virsh define /tmp/${NAME}.xml
virsh start ${NAME}
echo "Clone '${NAME}' created and started."
SCRIPT
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/make-clone.sh
```

Sanity-check:

```bash
sudo bash -n /usr/local/bin/make-clone.sh
```

No output = ✅ valid.

---

## 🚀 Recurring Workflow — Create a New Linked Clone

Always run these from the **[HOST]** prompt (`[appsquadz@cloudstack ~]$`).

### 1. Confirm you're on the host

```bash
hostname
```

Must say `cloudstack.local`.

### 2. Verify golden images exist

```bash
sudo ls -lh /var/lib/libvirt/images/golden-iphone*.qcow2
```

### 3. Create the clone (one command)

```bash
sudo make-clone.sh <clone-name>
```

**Example:**

```bash
sudo make-clone.sh iphone-clone-02
sudo make-clone.sh iphone-clone-03
sudo make-clone.sh test-vm-01
```

### 4. Verify it's running

```bash
sudo virsh list --all | grep <clone-name>
```

### 5. Verify it's a linked clone (no full copy)

```bash
sudo qemu-img info -U /var/lib/libvirt/images/<clone-name>.qcow2
```

Expected: `disk size: ~200 KB` and a `backing file:` line pointing to `golden-iphone.qcow2`.

### 6. Connect to the clone

```bash
sudo virsh console <clone-name>
```

Press **Enter**, log in. Exit with `Ctrl + ]`.

### 7. (Inside the clone) Post-boot cleanup if needed

**[VM]** — only if the golden image was not sysprepped:

```bash
sudo hostnamectl set-hostname <clone-name>
sudo truncate -s 0 /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo reboot
```

If cloud-init is installed, it may do this automatically on first boot.

---

1 min |
