+++
title = "KVM + Libvirt Setup Guide (with Cloud-Init and Bridged Networking)"
date = "2025-11-16T20:30:25-06:00"
draft = false
tags = ["kvm", "libvirt", "qemu", "cloud-init", "virtualization", "ubuntu", "homelab"]
categories = ["Homelab", "DevOps"]
description = "A guide to creating a bridged-network virtual machine using KVM, QEMU, Libvirt, and Cloud-Init on Ubuntu 24.04."
layout = "post"
+++

This guide outlines a reproducible, step-by-step workflow for provisioning a virtual machine using KVM, QEMU, Libvirt, and Cloud-Init on Ubuntu 24.04. The setup includes configuring a bridged network for LAN access, creating cloud-init files for initial user configuration, and managing the VM using `virt-install` and SSH.

---

## 1. Install KVM, QEMU, and Libvirt

Start by updating your system and installing required virtualization tools:

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst bridge-utils cloud-image-utils
```

Add your user to the necessary groups:

```bash
sudo usermod -aG libvirt,kvm $USER
```

> Log out and back in (or reboot) to apply group changes.

Verify virtualization support:

```bash
lscpu | grep -i virtualization
lsmod | grep kvm
qemu-system-x86_64 --version
```

---

## 2. Prepare Directories and Download Cloud Image

```bash
mkdir -p ~/muffins/{images,cloudinit}
```

Navigate and download the Ubuntu cloud image:

```bash
cd ~/muffins/images
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img -O ubuntu-24.04-cloudimg.qcow2
```

Create a working copy for the VM:

```bash
cp ubuntu-24.04-cloudimg.qcow2 muffins-vm01.qcow2
```

---

## 3. Configure Bridged Networking Using Netplan

If your system is using **NetworkManager** as the renderer (common on Ubuntu Desktop or systems with GUI), you can configure a network bridge using Netplan with NetworkManager support.

### 3.1 Create the Netplan file

```bash
sudo vim /etc/netplan/01-br0.yaml
```

Paste the following (update `eno1` if needed):

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
  bridges:
    br0:
      interfaces: [eno1]
      dhcp4: true
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

### 3.2 Apply changes

```bash
sudo netplan apply
```

### 3.3 Verify network

```bash
ip a
bridge link
```

Expected: `eno1` has no IP, `br0` receives a LAN IP.

---

## 4. Adjust Permissions for Libvirt

By default, Libvirt runs its QEMU processes under a dedicated system user (typically `libvirt-qemu` on Ubuntu). That user needs permission to:

- Traverse your home directory path (for example `/home/abraham`)
- Read the VM disk image and cloud-init ISO under `~/muffins`

If these permissions are missing, you will see errors when creating or starting the VM, such as:

- `Could not open disk image`
- `Permission denied` on the qcow2 file or seed ISO

Instead of loosening permissions on your entire home directory with `chmod`, you can use **POSIX Access Control Lists (ACLs)** to grant just enough access to the `libvirt-qemu` user.

### 4.1 Grant Execute Permission on Your Home Directory

This allows the `libvirt-qemu` user to traverse `/home/$USER` without granting read access to all files:

```bash
sudo setfacl -m u:libvirt-qemu:--x /home/$USER
```

The `--x` means:

- No read (`r`) or write (`w`) permission
- Execute (`x`) only, which for directories means “can enter / traverse this directory”

### 4.2 Grant Read/Execute on the VM Directory

Now allow Libvirt to read files under your `~/muffins` directory, where images and cloud-init files live:

```bash
sudo setfacl -R -m u:libvirt-qemu:r-x /home/$USER/muffins
```

Here `r-x` means:

- Read (`r`) allowed
- Execute (`x`) allowed (traverse directories)
- No write (`w`) permission

The `-R` flag applies this ACL recursively to the directory tree.

### 4.3 What Effect This Has

After these commands:

- Libvirt can **see and read**:
  - `~/muffins/images/muffins-vm01.qcow2`
  - `~/muffins/cloudinit/muffins-vm01-seed.iso`
- Libvirt **cannot modify** those files (no write permission granted).
- Your other directories under `$HOME` are not globally opened up; access is limited to what you explicitly granted with ACLs.

This strikes a balance between:

- Making Libvirt work reliably with VM files stored in your home directory, and
- Avoiding overly permissive `chmod 777`-style changes on `$HOME`.

---

## 5. Create Cloud-Init Configuration

Cloud-Init allows you to pass configuration to the VM at first boot, including the hostname, authorized SSH keys, and default user account.

You'll create two files in your `~/muffins/cloudinit` directory:

- `user-data` — contains Cloud-Init user setup instructions
- `meta-data` — provides instance metadata such as the hostname and ID

These files will be packaged into a seed ISO for the VM to read during boot.

---

### 5.1 Create `user-data`

Navigate to the Cloud-Init folder and create the `user-data` file:

```bash
cd ~/muffins/cloudinit
vim user-data
```

Enter insert mode (`i`) and paste the following:

```yaml
#cloud-config
hostname: muffins-vm01
users:
  - name: abraham
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    home: /home/abraham
    shell: /bin/bash
    lock_passwd: false
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...yourkeyhere
chpasswd:
  expire: false
```

Save and exit: press `Esc`, type `:wq`, and press `Enter`.

---

### 5.2 Create `meta-data`

Create the `meta-data` file:

```bash
vim meta-data
```

Enter insert mode and paste:

```yaml
instance-id: muffins-vm01
local-hostname: muffins-vm01
```

Save and exit with `Esc`, then `:wq`.

---

### 5.3 Build the Seed ISO

Use `cloud-localds` to generate the seed ISO from your configuration files:

```bash
sudo cloud-localds muffins-vm01-seed.iso user-data meta-data
sudo chown libvirt-qemu:kvm muffins-vm01-seed.iso
sudo chmod 640 muffins-vm01-seed.iso
```

The resulting `muffins-vm01-seed.iso` will be used during VM creation to apply your initial configuration.

---

## 6. Create the Virtual Machine with `virt-install`

With your system prepared, cloud-init seed ISO created, and bridged networking configured, you're now ready to create the virtual machine using `virt-install`.

`virt-install` is a command-line tool for creating VMs with Libvirt. It allows you to define CPU, memory, storage, network, and boot parameters in one command—making it ideal for provisioning repeatable VM environments.

---

### 6.1 Run `virt-install`

Run the following command to define and start the VM:

```bash
sudo virt-install \
  --name muffins-vm01 \
  --ram 4096 \
  --vcpus 2 \
  --disk path=/home/$USER/muffins/images/muffins-vm01.qcow2,format=qcow2 \
  --disk path=/home/$USER/muffins/cloudinit/muffins-vm01-seed.iso,device=cdrom \
  --os-variant ubuntu24.04 \
  --network bridge=br0,model=virtio \
  --graphics none \
  --import
```

---

### 6.2 Verify VM Creation

Once `virt-install` completes, verify that your VM is running:

```bash
sudo virsh list --all
```

If the VM isn’t running, inspect its logs:

```bash
sudo virsh console muffins-vm01
```

Use `Ctrl + ]` to exit the console.

---

### 6.3 Common `virt-install` Troubleshooting

- **Disk permission errors**:  
  Make sure the cloud-init ISO and qcow2 image are readable by `libvirt-qemu` (see Section 4)

- **Bridge not found**:  
  Check your network bridge exists with `ip a` and ensure its name matches (`br0`)

- **Unsupported OS variant**:  
  List supported variants with:

  ```bash
  osinfo-query os | grep ubuntu24
  ```

- **Start VM manually**:  
  If the VM wasn’t started automatically, run:

  ```bash
  sudo virsh start muffins-vm01
  ```

---

## 7. Access and Verify the Virtual Machine

Once your VM has been created using `virt-install`, the next step is to verify that it is running correctly and that you can access it.

---

### 7.1 Check VM Status with `virsh`

Use the `virsh` tool to view the list of running and stopped virtual machines:

```bash
sudo virsh list --all
```

You should see an entry similar to:

```
Id   Name          State
-----------------------------
 1   muffins-vm01  running
```

To view interfaces associated with the VM:

```bash
sudo virsh domiflist muffins-vm01
```

This command confirms that your VM is connected to the correct network (in this case, bridge `br0`).

---

### 7.2 Access the Console (Optional)

You can open a serial console to verify boot messages or debug issues:

```bash
sudo virsh console muffins-vm01
```

To exit the console, press:

```
Ctrl + ]
```

This is useful if:

- Cloud-init appears not to have run
- SSH is not responding
- The VM does not acquire a network lease

---

### 7.3 Find the VM's IP Address

Check `cloud-init` output during console login — it usually prints the network configuration. If not, you can:

- **Check DHCP leases on your router**
- **Use `virsh` to inspect guest info:**

  ```bash
  sudo virsh domifaddr muffins-vm01
  ```

  (Note: This works only if the guest agent is installed.)

- **Scan the local network with `nmap`:**

  ```bash
  sudo nmap -sn 192.168.2.0/24
  ```

  Look for a device with the hostname `muffins-vm01` or a newly discovered IP.

---

### 7.4 SSH Into the Virtual Machine

Once you find the VM's IP address, you can SSH into it:

```bash
ssh abraham@<vm-ip> -i ~/.ssh/id_ed25519
```

Replace `<vm-ip>` with the VM's actual IP address.

If using a custom SSH key or password login is enabled in cloud-init, adjust your connection command as needed.

---

## Summary

In this guide, you set up a virtual machine on Ubuntu 24.04 using KVM, QEMU, and Libvirt with the following key components:

- **Bridged networking** using Netplan, enabling your VM to behave like a physical device on your LAN.
- **Cloud-Init configuration** to define a hostname, create a user with SSH access, and configure the VM at first boot.
- **ISO-based provisioning** using `cloud-localds` to build a seed image for cloud-init.
- **VM creation** via `virt-install`, specifying CPU, memory, storage, network, and initial boot parameters.
- **Access and verification** using `virsh` and SSH.

This workflow provides a reproducible and flexible method for building VMs suitable for DevOps testing, Kubernetes labs, or application deployments. With bridge networking and cloud-init in place, you can easily clone or automate this environment for larger-scale projects.
