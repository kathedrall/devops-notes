# KVM/Libvirt Swiss Army Knife
## Complete Guide for Ubuntu/Debian Systems

[![KVM](https://img.shields.io/badge/KVM-Virtualization-blue)](https://www.linux-kvm.org/)
[![Libvirt](https://img.shields.io/badge/Libvirt-Management-green)](https://libvirt.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20LTS-orange)](https://ubuntu.com/)

---

## Table of Contents

1. [Downloading Cloud Images](#1-downloading-cloud-images)
2. [Host Server Preparation](#2-host-server-preparation)
3. [Disk Preparation](#3-disk-preparation)
4. [Offline Customization (virt-customize)](#4-offline-customization-virt-customize)
5. [SSH Configuration & Security Hardening](#5-ssh-configuration--security-hardening)
6. [Network Configuration (Netplan)](#6-network-configuration-netplan)
7. [VM Creation (virt-install)](#7-vm-creation-virt-install)
8. [Post-Installation Tasks](#8-post-installation-tasks)
9. [SSH Key Management](#9-ssh-key-management)
10. [Useful Commands & Troubleshooting](#10-useful-commands--troubleshooting)

---

## 1. Downloading Cloud Images

Download official Ubuntu cloud images from:
```
https://cloud-images.ubuntu.com/
```

> **Tip:** Prefer `.img` or `.qcow2` image formats for compatibility.

---

## 2. Host Server Preparation

### Check Virtualization Support

Verify if CPU virtualization is enabled:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

**Result interpretation:**
- **0** = Virtualization disabled → Reboot, enable Intel VT-x or AMD-V in BIOS/UEFI
- **> 0** = Virtualization enabled → Proceed with installation

### Install Required Packages

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst libguestfs-tools
```

---

## 3. Disk Preparation

> **Important:** Never use the original downloaded image directly. Always create a working copy.

### 1. Copy Base Image

```bash
cd ~/Downloads/images/
sudo cp jammy-server-cloudimg-amd64.disk.img /var/lib/libvirt/images/vm-jammy.qcow2
```

### 2. Resize Disk

Example: Increase to 10GB

```bash
sudo qemu-img resize /var/lib/libvirt/images/vm-jammy.qcow2 10G
```

---

## 4. Offline Customization (virt-customize)

Configure password, remove cloud-init, and inject SSH keys **WITHOUT** starting the VM.

### Command 1: Set Passwords & Remove Cloud-Init (Essential)

```bash
sudo virt-customize -a /var/lib/libvirt/images/vm-jammy.qcow2 \
  --root-password password:123456 \
  --uninstall cloud-init \
  --password ubuntu:password:123456
```

### Command 2: Inject SSH Public Key (Optional)

Replace with your actual public key path:

```bash
sudo virt-customize -a /var/lib/libvirt/images/vm-jammy.qcow2 \
  --ssh-inject ubuntu:file:/home/samuel/.ssh/id_rsa.pub
```

**Or** using a PEM key:

```bash
sudo virt-customize -a /var/lib/libvirt/images/vm-jammy.qcow2 \
  --ssh-inject ubuntu:file:/home/samuel/.ssh/chave-vm3.pem.pub
```

---

## 5. SSH Configuration & Security Hardening

### Option A: Block Root Login, Allow Password for Regular User

Use this when you want to login with password (e.g., ubuntu/123456) but prevent root direct login.

```bash
sudo virt-customize -a /var/lib/libvirt/images/vm-jammy.qcow2 \
  --edit '/etc/ssh/sshd_config:
    s/^#?PermitRootLogin.*/PermitRootLogin no/;
    s/^#?PasswordAuthentication.*/PasswordAuthentication yes/'
```

### Option B: SSH Key Only (No Password, No Root)

> **Warning:** Inject SSH key BEFORE running this command!

```bash
sudo virt-customize -a /var/lib/libvirt/images/vm-jammy.qcow2 \
  --edit '/etc/ssh/sshd_config:
    s/^#?PermitRootLogin.*/PermitRootLogin no/;
    s/^#?PasswordAuthentication.*/PasswordAuthentication no/;
    s/^#?ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/;
    s/^#?PubkeyAuthentication.*/PubkeyAuthentication yes/'
```

> **Note:** On some Ubuntu versions, you may need to inject configuration into `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf`. If customization doesn't work on `/etc/ssh/sshd_config`, repeat the commands for the alternative path.

---

## 6. Network Configuration (Netplan)

Eliminates the need to manually configure IP inside the VM. This command writes the YAML file directly to disk.

```bash
sudo virt-customize -a /var/lib/libvirt/images/vm-jammy.qcow2 \
  --hostname vm-jammy \
  --uninstall cloud-init \
  --write '/etc/netplan/01-netcfg.yaml:
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: no
      addresses:
        - 192.168.122.55/24
      routes:
        - to: default
          via: 192.168.122.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
'
```


---

## 7. VM Creation (virt-install)

Command to create and start the VM in Libvirt:

```bash
sudo virt-install \
  --name vm-jammy \
  --memory 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/vm-jammy.qcow2,device=disk,bus=virtio \
  --os-variant ubuntu22.04 \
  --import \
  --network network=default,model=virtio \
  --graphics none
```

---

## 8. Post-Installation Tasks

Access via SSH or Console to perform these final adjustments.

### SSH Access Methods

```bash
# With password
ssh ubuntu@192.168.122.55

# With SSH key
ssh ubuntu@192.168.122.55

# With PEM key
ssh -i /home/samuel/.ssh/chave-vm3.pem ubuntu@192.168.122.55
```

> **Tip:** If SSH access fails, use the console:
> ```bash
> sudo virsh console vm-jammy
> ```

### A) Expand Disk

```bash
sudo apt update && sudo apt install -y cloud-guest-utils
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
```

### B) Configure Network (Static IP)

> **Note:** If virt-customize network configuration didn't work, you can only access the VM via console to configure manually.

Edit `/etc/netplan/01-netcfg.yaml`:

#### Ubuntu 22.04+ Template

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: no
      addresses:
        - 192.168.122.55/24
      routes:
        - to: default
          via: 192.168.122.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

#### Ubuntu 18.04 Template

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: no
      addresses:
        - 192.168.122.55/24
      gateway4: 192.168.122.1
      nameservers:
        addresses: [8.8.8.8]
```

**Apply configuration:**
```bash
sudo netplan apply
```

### C) Fix SSH Issues

If you encounter "no hostkeys" errors or password authentication issues:

1. **Generate missing keys:**
   ```bash
   sudo ssh-keygen -A
   ```

2. **Enable password authentication** in `/etc/ssh/sshd_config`:
   ```
   PasswordAuthentication yes
   ```

3. **Restart SSH service:**
   ```bash
   sudo systemctl restart ssh
   ```

---

## 9. SSH Key Management

### A) Generate SSH Key on Host

```bash
ssh-keygen -t ed25519 -C "user@host"
```

### B) Copy Key to VM (Passwordless Login)

```bash
ssh-copy-id ubuntu@192.168.122.55
```

### C) Create PEM Key

1. **Generate PEM key:**
   ```bash
   ssh-keygen -t rsa -b 4096 -m PEM -f chave-vm.pem
   ```

2. **Inject public key into VM:**
   Use virt-customize (see section 4)

3. **Connect with PEM key:**
   ```bash
   ssh -i chave-vm.pem ubuntu@192.168.122.55
   ```

4. **Set correct permissions:**
   ```bash
   chmod 400 chave-vm.pem
   ```

---

## 10. Useful Commands & Troubleshooting

### VM Management

| Action | Command |
|--------|---------|
| **List VMs** | `sudo virsh list --all` |
| **Start VM** | `sudo virsh start VM_NAME` |
| **Shutdown (graceful)** | `sudo virsh shutdown VM_NAME` |
| **Force stop** | `sudo virsh destroy VM_NAME` |
| **Delete VM** | `sudo virsh undefine VM_NAME` |

> **Note:** `undefine` does NOT delete the disk image.

### Console Access

```bash
# Access console
sudo virsh console VM_NAME

# Exit console: Press CTRL + ]
```

**If console is already in use:**
```bash
sudo virsh console VM_NAME --force
```

### Network Troubleshooting

**Discover VM IP (if using DHCP):**
```bash
sudo virsh net-dhcp-leases default
```

---

## Summary

This cheatsheet provides a complete workflow for:
- Preparing KVM/Libvirt hosts
- Creating VMs from cloud images
- Offline customization (no boot required)
- Security hardening (SSH configuration)
- Network configuration
- VM lifecycle management

---

## Additional Resources

- [Official KVM Documentation](https://www.linux-kvm.org/)
- [Libvirt Documentation](https://libvirt.org/docs.html)
- [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/)
- [Netplan Reference](https://netplan.io/)

---

**Last Updated:** January 2026  
**Author:** Samuel  
**License:** MIT

---
