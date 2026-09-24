# Method 2: Full Clone — Simple Playbook

Make a **completely independent copy** of a KVM VM using `virt-clone`.

---

## What You Get

| | |
|---|---|
| **Result** | A new VM that shares nothing with the source |
| **Disk used** | Same as source (10 GB → 10 GB) |
| **Time to create** | ~90 seconds |
| **Depends on source?** | No — delete the source and the clone still works |

---

## Before You Start

Make sure you have:

- A **shut-off** source VM (we use `dell` as the example)
- Enough **free space** in `vg_vm` for a full copy of every disk
- The **host prompt** — `[appsquadz@cloudstack ~]$`

Check the source first:

```bash
sudo virsh domstate dell
```

Must say **`shut off`**. If not:

```bash
sudo virsh shutdown dell
```

See the source's disks:

```bash
sudo virsh domblklist dell --details
```

Note the **order** (vda first, vdb second). That order matters later.

Check free space:

```bash
sudo vgs
```

---

## Step-by-Step

### Step 1 — Create the destination disks

`virt-clone` will not create LVs for you. Do it manually.

```bash
sudo lvcreate -L 10G -n dell-fullclone-lv1 vg_vm
sudo lvcreate -L 10G -n dell-fullclone-lv2 vg_vm
```

Verify:

```bash
sudo lvs | grep dell-fullclone
```

You should see two empty 10 GB LVs.

---

### Step 2 — Run the clone

```bash
sudo virt-clone \
  --original dell \
  --name dell-fullclone \
  --file /dev/vg_vm/dell-fullclone-lv1 \
  --file /dev/vg_vm/dell-fullclone-lv2 \
  --check path_exists=off
```

**What each flag means:**

| Flag | Meaning |
|---|---|
| `--original dell` | Copy FROM this VM |
| `--name dell-fullclone` | Name of the NEW VM |
| `--file ...lv1` | New disk for vda |
| `--file ...lv2` | New disk for vdb |
| `--check path_exists=off` | Allow writing to pre-created LVs |

Takes about **90 seconds** for 20 GB total.

---

### Step 3 — Start the clone

```bash
sudo virsh start dell-fullclone
sudo virsh list --all | grep dell
```

Expected:

```text
 9    dell-fullclone    running
 -    dell              shut off
```

---

### Step 4 — Eject the autoinstall CDROM

The clone inherited the source's CDROM. If it boots from that ISO, it will **wipe the clone**. Remove it now:

```bash
sudo virsh change-media dell-fullclone sdb --eject --live --config
```

Verify:

```bash
sudo virsh domblklist dell-fullclone --details
```

The `sdb` row should show `-` for Source.

---

### Step 5 — Clean up inside the clone

The clone is a byte-for-byte copy. It has the **same hostname, machine-id, and SSH keys** as the source — which causes network conflicts. Fix inside the clone.

Connect:

```bash
sudo virsh console dell-fullclone
```

Log in, then run the block for your OS.

**Ubuntu / Debian:**

```bash
sudo hostnamectl set-hostname dell-fullclone
sudo truncate -s 0 /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo reboot
```

**RHEL / CentOS / Rocky / Alma:**

```bash
sudo hostnamectl set-hostname dell-fullclone
sudo truncate -s 0 /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo systemctl restart sshd
sudo reboot
```

Exit the console with `Ctrl + ]`.

---

## Verify It Worked

Check the clone is independent — no `s` (snapshot) flag, no origin:

```bash
sudo lvs | grep dell
```

Expected:

```text
dell                vg_vm -wi-a-----  10.00g
dell01              vg_vm -wi-a-----  10.00g
dell-fullclone-lv1  vg_vm -wi-ao----  10.00g
dell-fullclone-lv2  vg_vm -wi-ao----  10.00g
```

If `Origin` is blank and no `s` appears → **full clone, independent**. ✅

Check no KVM snapshots exist:

```bash
sudo virsh snapshot-list dell-fullclone
```

Empty table = no snapshots = true full clone.

---

## Common Commands

| Task | Command |
|---|---|
| Stop the clone | `sudo virsh shutdown dell-fullclone` |
| Start the clone | `sudo virsh start dell-fullclone` |
| Show clone's disks | `sudo virsh domblklist dell-fullclone --details` |
| Show clone's info | `sudo virsh dominfo dell-fullclone` |
| Delete the clone | See below |

Delete the clone completely:

```bash
sudo virsh destroy dell-fullclone 2>/dev/null
sudo virsh undefine dell-fullclone
sudo lvremove -f /dev/vg_vm/dell-fullclone-lv1
sudo lvremove -f /dev/vg_vm/dell-fullclone-lv2
```

---

## Full Clone vs Linked Clone

| | Full Clone | Linked Clone |
|---|---|---|
| Disk used | ~20 GB | ~200 KB |
| Time | ~90 sec | ~2 sec |
| Independent? | ✅ Yes | ❌ Needs base image |
| Portable? | ✅ Yes | ❌ No |
| Best for | Standalone VMs | Fast test VMs |

---

## Rules to Remember

1. **Shut off the source** before cloning.
2. **Pre-create the destination LVs** — `virt-clone` won't.
3. **Match `--file` order** to `vda` / `vdb` order.
4. **Always eject the autoinstall CDROM** after cloning.
5. **Change hostname / machine-id / SSH keys** inside the clone.
6. **Never delete the source LVs** (`dell`, `dell01`).
7. **Run everything from the HOST**, not inside a VM.

---

## Host vs VM — Quick Check

Not sure where you are? Run:

```bash
hostname
```

| Output | You're on |
|---|---|
| `cloudstack.local` | **HOST** — run `virsh`, `virt-clone`, `lvcreate` |
| Anything else | **VM** — run `hostnamectl`, `dpkg`, `ssh-keygen` |

Exit any VM console with **`Ctrl + ]`**.
