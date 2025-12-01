+++
date = '2025-11-29T19:45:42-06:00'
draft = false
title = 'Setup KVM via Terraform (with Cloud-Init and Bridged Networking)'
+++

# **Terraform Bring-Up Adventures**  
_This write up is a continuation of the prior post where I create a fully reproducible enviroment to deploy a 3-node lab using QEMU/KVM, libvirt, Terraform, cloud-init, AppArmor tuning, and sysctl kernel configuration._

---

# **1. Overview**

TXGrid is a reproducible 3-node lab environment that runs entirely on your workstation using:

- QEMU/KVM via libvirt  
- Netplan-hosted Linux bridge for LAN access  
- A private libvirt network for cluster traffic  
- Terraform provisioning  
- Cloud-init for automation  
- Sysctl & AppArmor adjustments to support routing & QEMU access  

This runbook documents the additional steps from prior post needed to successfully bring up:

- `txgrid-cp0` (control plane)
- `txgrid-wk0` (worker 0)
- `txgrid-wk1` (worker 1)

All nodes run **Ubuntu Server 24.04 cloud image**.

---

# **2. Prerequisites & Host Preparation**

Refer to the prior blog post [kvm_libvirt_setup_guide](https://mrmcmuffinz.github.io/posts/kvm_libvirt_setup_guide/) for:

- Installation of QEMU/KVM, libvirt, and supporting packages
- Bridge creation via Netplan
- User/group permissions for libvirt and KVM
- Cloud-init basics

This guide focuses **only on what is required in addition** to the earlier setup.

---

# **3. Architecture Overview**

TXGrid uses:

- A Linux bridge (`br0`) for external network access  
- A libvirt-managed internal network for inter-node traffic  
- Terraform to orchestrate all VMs, disks, NICs, and cloud-init configurations  
- Cloud-init for hostname assignment, SSH access, package installs, and bootstrap scripting  

---

# **4. Terraform Bring-Up (High Level)**

Terraform handles:

- Volume creation (qcow2 cloning)
- Domain definitions for each VM
- NIC attachment (bridge + private network)
- Cloud-init ISO injection
- VM boot ordering and metadata

The full configs and modules are in the GitHub [my-ai-journey/tree/v0.1.1/infra/terraform](https://github.com/mrmcmuffinz/my-ai-journey/tree/v0.1.1/infra/terraform). The code is under development and constantly changing however I have linked the version v0.1.1 that I wrote for this runbook.

---

# **5. AppArmor Adjustments**

Ubuntu’s default AppArmor profiles may block certain QEMU operations, especially when:

- Using bridged networking
- Accessing custom disk paths or cloud-init data
- Running Terraform-created domains that reference nonstandard directories

Symptoms include:

- `qemu-system: failed to open /dev/...`
- AppArmor DENIED messages in `dmesg` or `/var/log/syslog`
- Terraform-created domains failing to launch

This resolves `Permission denied` issues when QEMU attempts to read Terraform-created files:

Create the folder 

```bash
sudo mkdir -p /etc/apparmor.d/abstractions/libvirt-qemu.d/
```

Edit: `/etc/apparmor.d/abstractions/libvirt-qemu.d/override`

Add lines (The comma at the end is important do not omit it!):

```bash
/var/lib/libvirt/images/** rwk,
```

Reload AppArmor and libvirtd:

```bash
sudo systemctl restart apparmor
sudo systemctl restart libvirtd
```

Check status:

```bash
sudo aa-status|grep libvirtd
```

You should see libvirt-related profiles in **enforce** mode, with your updated rules in effect.

```bash
   libvirtd
   libvirtd//qemu_bridge_helper
   /usr/sbin/libvirtd (50117) libvirtd
```

---

# **6. Kernel Tuning (sysctl)**

These settings are required to support forwarding and inter-VM routing. Without the kernel parameter enabled the bridged adapter for each node will not receive an ip.

- Allow IPv4 forwarding between `br0` and `txgrid-net`

Modify:

```bash
sudo echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
```

Apply:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

# **7. Migrating to the Latest Terraform libvirt Provider**

**Status:** Migration is _work in progress_. I created a WIP branch with partial changes, but I did **not** finish migrating to the latest provider due to upstream bugs and instability. I am **not** recommending an upgrade at this time.

WIP-Code: [my-ai-journey/tree/upgrade_libvirt_provider_to_0_9_x](https://github.com/mrmcmuffinz/my-ai-journey/tree/upgrade_libvirt_provider_to_0_9_x)  
I also have a discussion open with the maintainer as well [terraform-provider-libvirt/discussions/1231](https://github.com/dmacvicar/terraform-provider-libvirt/discussions/1231) however time will tell if upstream will stabilize or not.

Observed issues during attempted migration to 0.9.0 (and why I paused):

- Breaking changes in resource attributes that required nontrivial refactors of node modules.  
- Removal or renaming of disk/network attributes that made old state incompatible.  
- Domain XML differences and provider behavior changes causing Terraform to attempt destructive updates.  
- Upstream provider bugs causing intermittent failures during `plan`/`apply`.  
- State files from older versions becoming difficult to reconcile without careful manual interventions.

Current guidance:

- Continue using the provider version that matches your working branch / environment.  
- If you experiment with the WIP branch, do so on disposable state and snapshots only.  
- Watch upstream provider issue tracker for patches/fixes before attempting production migration.  
- I have preserved my WIP branch in the repo for anyone who wants to inspect the attempted changes, but **do not** treat it as stable or recommended.

---

# **8. Operation Workflow**

Steps to run the code are documented in the repo, at a high level these are the general steps:

1. Clone the code.
2. navigate to infra/terraform folder.
3. Run a `terraform init` to install all dependencies.
4. Run `terraform plan` to preview the changes.
5. Run `terraform apply -auto-approve` to provision the infrastructure.

**Note:** You have to run step 5 twice due to a bug in the terraform provider that doesn't refresh the internal state of libvirt. What ends up happening is that the networking information is stale and on a second run it gets populated.  

Here is what the outputs look like

```bash
Outputs:

vm_ip_addresses = [
  {
    "txgrid-cp0" = tolist([
      {
        "addresses" = tolist([
          "192.168.2.166",
          "fd8c:9056:52a4:f645:3697:f6ff:feaa:bbc0",
          "fe80::3697:f6ff:feaa:bbc0",
        ])
        "bridge" = "br0"
        "hostname" = ""
        "mac" = "34:97:F6:AA:BB:C0"
        "macvtap" = ""
        "network_id" = ""
        "network_name" = ""
        "passthrough" = ""
        "private" = ""
        "vepa" = ""
        "wait_for_lease" = false
      },
      {
        "addresses" = tolist([
          "192.168.50.10",
          "fe80::5054:ff:fe5e:144e",
        ])
        "bridge" = ""
        "hostname" = ""
        "mac" = "52:54:00:5E:14:4E"
        "macvtap" = ""
        "network_id" = "82dc306f-1a38-4e32-b944-f5c8ad31351d"
        "network_name" = "hostnet"
        "passthrough" = ""
        "private" = ""
        "vepa" = ""
        "wait_for_lease" = false
      },
    ])
  },
  {
    "txgrid-wk1" = tolist([
      {
        "addresses" = tolist([
          "192.168.2.167",
          "fd8c:9056:52a4:f645:3697:f6ff:feaa:bbc1",
          "fe80::3697:f6ff:feaa:bbc1",
        ])
        "bridge" = "br0"
        "hostname" = ""
        "mac" = "34:97:F6:AA:BB:C1"
        "macvtap" = ""
        "network_id" = ""
        "network_name" = ""
        "passthrough" = ""
        "private" = ""
        "vepa" = ""
        "wait_for_lease" = false
      },
      {
        "addresses" = tolist([
          "192.168.50.11",
          "fe80::5054:ff:febe:ee14",
        ])
        "bridge" = ""
        "hostname" = ""
        "mac" = "52:54:00:BE:EE:14"
        "macvtap" = ""
        "network_id" = "82dc306f-1a38-4e32-b944-f5c8ad31351d"
        "network_name" = "hostnet"
        "passthrough" = ""
        "private" = ""
        "vepa" = ""
        "wait_for_lease" = false
      },
    ])
  },
  {
    "txgrid-wk2" = tolist([
      {
        "addresses" = tolist([
          "192.168.2.168",
          "fd8c:9056:52a4:f645:3697:f6ff:feaa:bbc2",
          "fe80::3697:f6ff:feaa:bbc2",
        ])
        "bridge" = "br0"
        "hostname" = ""
        "mac" = "34:97:F6:AA:BB:C2"
        "macvtap" = ""
        "network_id" = ""
        "network_name" = ""
        "passthrough" = ""
        "private" = ""
        "vepa" = ""
        "wait_for_lease" = false
      },
      {
        "addresses" = tolist([
          "192.168.50.12",
          "fe80::5054:ff:fe9a:3209",
        ])
        "bridge" = ""
        "hostname" = ""
        "mac" = "52:54:00:9A:32:09"
        "macvtap" = ""
        "network_id" = "82dc306f-1a38-4e32-b944-f5c8ad31351d"
        "network_name" = "hostnet"
        "passthrough" = ""
        "private" = ""
        "vepa" = ""
        "wait_for_lease" = false
      },
    ])
  },
]
```

# **9. Verification Steps**

At this point the nodes should be online and below are a set of steps you can run to verify a few things.

Verify domains:

```bash
virsh list --all
```

You should see:

```bash
 Id   Name         State
----------------------------
 1    txgrid-wk2   running
 2    txgrid-cp0   running
 3    txgrid-wk1   running
```

all in the `running` state.

You can also get VM IP addresses via QEMU guest agent using the domain name.


```bash
virsh domifaddr --source agent txgrid-cp0
```

Example successful output (for `txgrid-cp0`):

```bash
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 lo         00:00:00:00:00:00    ipv4         127.0.0.1/8
 -          -                    ipv6         ::1/128
 ens3       34:97:f6:aa:bb:c0    ipv4         192.168.2.166/24
 -          -                    ipv6         fd8c:9056:52a4:f645:3697:f6ff:feaa:bbc0/64
 -          -                    ipv6         fe80::3697:f6ff:feaa:bbc0/64
 ens4       52:54:00:5e:14:4e    ipv4         192.168.50.10/24
 -          -                    ipv6         fe80::5054:ff:fe5e:144e/64
```

SSH into txgrid-cp0:

```bash
ssh -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no txgrid@192.168.50.10
```

Output:

```bash
Warning: Permanently added '192.168.50.10' (ED25519) to the list of known hosts.
Welcome to Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-88-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Nov 30 04:04:31 UTC 2025

  System load:           0.0
  Usage of /:            94.6% of 2.35GB
  Memory usage:          2%
  Swap usage:            0%
  Processes:             135
  Users logged in:       0
  IPv4 address for ens3: 192.168.2.166
  IPv6 address for ens3: fd8c:9056:52a4:f645:3697:f6ff:feaa:bbc0

  => / is using 94.6% of 2.35GB


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Sun Nov 30 04:04:31 2025 from 192.168.50.1
txgrid@cp0:~$ uptime
 04:05:40 up 31 min,  1 user,  load average: 0.00, 0.00, 0.00
txgrid@cp0:~$
```


Connectivity tests between nodes:

From **txgrid-cp0 -> txgrid-wk1**:

```bash
txgrid@cp0:~$ ping -c 3 192.168.50.11
PING 192.168.50.11 (192.168.50.11) 56(84) bytes of data.
64 bytes from 192.168.50.11: icmp_seq=1 ttl=64 time=0.177 ms
64 bytes from 192.168.50.11: icmp_seq=2 ttl=64 time=0.203 ms
64 bytes from 192.168.50.11: icmp_seq=3 ttl=64 time=0.261 ms

--- 192.168.50.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2037ms
rtt min/avg/max/mdev = 0.177/0.213/0.261/0.035 ms
```

From **txgrid-cp0 -> txgrid-wk2**:

```bash
txgrid@cp0:~$ ping -c 3 192.168.50.12
PING 192.168.50.12 (192.168.50.12) 56(84) bytes of data.
64 bytes from 192.168.50.12: icmp_seq=1 ttl=64 time=0.277 ms
64 bytes from 192.168.50.12: icmp_seq=2 ttl=64 time=0.236 ms
64 bytes from 192.168.50.12: icmp_seq=3 ttl=64 time=0.232 ms

--- 192.168.50.12 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2053ms
rtt min/avg/max/mdev = 0.232/0.248/0.277/0.020 ms
```

Connectivity outbound request to public:

```
txgrid@cp0:~$ ping -c 3 www.google.com
PING www.google.com (172.253.124.99) 56(84) bytes of data.
64 bytes from ys-in-f99.1e100.net (172.253.124.99): icmp_seq=1 ttl=104 time=40.1 ms
64 bytes from ys-in-f99.1e100.net (172.253.124.99): icmp_seq=2 ttl=104 time=40.1 ms
64 bytes from ys-in-f99.1e100.net (172.253.124.99): icmp_seq=3 ttl=104 time=40.6 ms

--- www.google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 40.067/40.237/40.559/0.227 ms
```

---

# **10. Troubleshooting & Gotchas**

### **Common Issues**

| Issue | Likely Cause | Fix |
|-------|--------------|------|
| VMs fail to start | AppArmor restriction | Update QEMU profile (see repo diffs) or relax policy for lab usage |
| NIC missing | Bridge misconfiguration | Validate Netplan & ensure bridge exists before VM start |
| Cloud-init not applying | Wrong metadata or template path | Rebuild the cloud-init ISO module |
| Terraform repeatedly recreates resources | Provider incompatibility | Use pinned provider version and avoid migrating until upstream stabilizes |

### **Useful Commands**

```bash
virsh list --all
virsh domifaddr txgrid-cp0
virsh net-dhcp-leases default
virsh dumpxml txgrid-wk1
```

---

# **11. Further Enhancements**

Future improvements may include:

- Ansible provisioning post-boot
- GPU passthrough testing for ML workloads
- iperf-based network benchmarking between nodes
