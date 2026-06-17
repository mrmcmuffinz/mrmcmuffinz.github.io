+++
date = '2026-06-17T08:22:28-05:00'
draft = false
title = 'Building a 3-Node Raspberry Pi 5 Cluster (Before Kubernetes)'
tags = ["raspberry-pi", "kubernetes", "homelab", "cloud-init", "debian", "networking", "vlan", "ssh"]
categories = ["homelab"]
description = "Bootstrapping a 3-node Raspberry Pi 5 cluster: image patching with bake.sh, cloud-init provisioning, an isolated VLAN, and the gotchas along the way, as groundwork for a future Kubernetes build."
layout = "post"
+++

# Building a 3-Node Raspberry Pi 5 Cluster (Before Kubernetes)

Welcome back friends. I've been working toward the Certified Kubernetes Administrator exam, and for a while I was happy enough practicing on virtual machines. At some point I wanted to test on real hardware, but I also didn't want a giant pile of nodes humming away in my closet, so I settled on a three-node Raspberry Pi cluster. This post is about everything that happens before Kubernetes ever enters the picture: assembling the Pis, patching the OS image, driving the first boot with cloud-init, flashing the storage, and getting the network set up so I can actually reach the nodes. I'll cover the Kubernetes install itself in a separate post, because honestly that deserves its own writeup. If you'd rather watch me walk through the whole thing, the video version is below.

{{< youtube "dgH-cxxOXy8" >}}

Here's the end-to-end flow at a glance, from downloading the image to verifying a running node:

{{< mermaid >}}
flowchart TD
    A[Download Pi OS Trixie Lite arm64] --> B[Run bake.sh to produce patched image]
    B --> D[Flash patched image to NVMe via enclosure]
    D --> E[Mount boot partition]
    E --> F[Write per-node user-data, network-config, meta-data]
    F --> G[Unmount and install NVMe in Pi]
    G --> H[Configure the network]
    H --> I[Power on the Pi, cloud-init runs on first boot]
    I --> J[SSH in and verify configuration]
{{< /mermaid >}}

## Why Debian Trixie (and Not Ubuntu)

I went with Raspberry Pi OS Lite, which is Debian 13 Trixie under the hood, and that wasn't an accident. When I first started this project I actually used Ubuntu 24.04, but the resource utilization on the Pi was noticeably higher than Debian for what is essentially the same job. Trixie is the latest stable Debian, it ships a smaller image, and it runs leaner on a board where every bit of memory matters. There's a security angle too, since fewer things installed and running means less surface area for someone to attack, and that lines up with how I want to think about a cluster rather than a toy. It's also close enough to what the CKA exam environment looks like that I'm not learning habits I'll have to unlearn later. Less running, less installed, closer to the target, that was the whole reasoning.

## The Hardware

My router is a UniFi Cloud Gateway Fiber, and there's a UniFi Switch 24 sitting between it and my office. The switch isn't really important to the story, it's just where everything physically lands, but it's the older revision that still has the internal fans if you care about that sort of thing. My Ubuntu 24.04 desktop is the host I use to flash the drives and to SSH into the cluster, and it has five network ports total: one on the motherboard plus a quad-port PCIe card, which matters later when I carve out a dedicated link for the Pis.

The cluster itself is three Raspberry Pi 5 boards with 8GB of RAM each, and they're not just bare boards sitting on my desk. Every one of them has an active-cooling heatsink with an integrated fan, and all three live in metal enclosures so the whole thing feels like actual gear rather than a science fair project. Each Pi boots from its own NVMe drive, connected through a small adapter board and ribbon cable rather than a traditional HAT, and to flash those drives from my desktop I use a single USB-to-NVMe enclosure that I pop each drive into one at a time.

## Patching the Image with bake.sh

The stock Pi OS image isn't quite ready for what I want, so I wrote a script called `bake.sh` that takes the image I downloaded and produces a ready-to-flash patched copy. It accepts both `.img` and `.img.xz` inputs and always works on a copy, so the original I downloaded is never touched.

```bash
sudo ./bake.sh 2026-04-21-raspios-trixie-arm64-lite.img.xz
```

Under the hood it decompresses the image if needed, mounts it through a loop device on my desktop, and makes a handful of changes that I'd otherwise have to repeat by hand on every node. It appends the cgroup flags to `cmdline.txt` that Kubernetes needs for memory accounting, enables PCIe gen 3 in `config.txt` so the NVMe drives run at full throughput, and disables swap. Swap on the Pi isn't the apples-to-apples swapfile you'd expect coming from an x86 desktop, since Trixie uses a zram-based generator, but the point is the same: Kubernetes doesn't want swap, so I make sure it's gone. The script also seeds placeholder cloud-init files onto the boot partition so the NoCloud datasource activates, and then it chroots into the ARM64 image through `qemu-aarch64-static` to preinstall the packages I know I'll want, harden SSH by disabling root login, and turn off the Pi OS first-boot wizard that otherwise races with cloud-init. Splitting the work this way, image-level changes in `bake.sh` and per-node changes later, is what made reflashing a node a two-minute affair instead of a chore.

## Cloud-Init: One Image, Three Nodes

Once the image is baked, the per-node differences come from three cloud-init files written to the boot partition: `meta-data`, `network-config`, and `user-data`. Cloud-init reads them through the NoCloud datasource on first boot, does its work, and reboots the node when it's done. The only things that actually change between nodes are the hostname and the IP address. The `meta-data` file just sets the instance ID and local hostname to match the node, and the `network-config` file pins a static address in Netplan v2 format, which NetworkManager applies on Trixie.

```yaml
version: 2
ethernets:
  eth0:
    dhcp4: false
    addresses:
      - 192.168.200.10/24
    routes:
      - to: default
        via: 192.168.200.1
    nameservers:
      addresses:
        - 192.168.200.1
        - 1.1.1.1
```

The `user-data` file does the heavier lifting, and it's mostly identical across all three nodes. It sets the timezone, forces the keyboard layout to US (which I had to track down, because the first boot kept defaulting to a Great Britain layout and I'm not in the UK), writes the kernel modules and sysctl settings Kubernetes will want later, and creates an `admin` user that can only get in over SSH with a key and has passwordless sudo. The most interesting part is a workaround I had to add for a problem that's specific to the Pi. The board has no real-time clock, so on first boot the system time is far enough off that apt rejects the package signatures it pulls down, which means a plain `apt-get update` simply fails. The fix is to force an NTP sync in `runcmd` and wait for the clock to actually catch up before touching apt at all.

```yaml
runcmd:
  # The Pi has no real-time clock, so NTP must sync before apt runs,
  # otherwise package signature verification fails.
  - timedatectl set-ntp true
  - |
    for i in $(seq 1 30); do
      if timedatectl show -p NTPSynchronized --value | grep -q yes; then
        break
      fi
      sleep 2
    done
  - apt-get update
```

The static addresses live entirely in these files rather than as reservations on the router, and that's deliberate, because the VLAN I built for the cluster doesn't run a DHCP server at all. The per-node mapping is small enough to keep in one place:

| Node   | hostname / instance-id | Address             |
| ------ | ---------------------- | ------------------- |
| Node 1 | `rpi-node-01`          | `192.168.200.10/24` |
| Node 2 | `rpi-node-02`          | `192.168.200.11/24` |
| Node 3 | `rpi-node-03`          | `192.168.200.12/24` |

## Flashing and the Per-Node Files

Flashing is the least exciting part, which is exactly how I like it. I drop an NVMe drive into the USB enclosure, identify it by size so I don't write to the incorrect device, and `dd` the patched image onto it.

```bash
# Identify the device by size first
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Then flash the patched image
sudo dd if=raspios-trixie-arm64-lite-patched.img of=/dev/sda bs=4M status=progress conv=fsync
sync
```

After the flash finishes I mount the boot partition, edit the hostname and IP in the three cloud-init files for that specific node, copy them over, and unmount. Then I repeat the whole thing for the next drive. It's a one-time bootstrap per node, and is there a slicker way to do it than mounting and editing by hand? Almost certainly, but this worked and I didn't want to gold-plate it. I'll admit the first node took more passes than I'd like too. I reflashed `rpi-node-01` at least five times while I was figuring out cloud-init and chasing down the keyboard, swap, and clock issues. By the time I got to the second node the config was mostly settled, and the third node ended up with the cleanest version of all, since the baked image was already handling the package work that earlier iterations were trying to do at boot.

## Assembling the Pis

Here's a single node laid out before assembly: the Pi 5 in the center, the active-cooling heatsink with its integrated fan, the NVMe drive, and the adapter board that lets the Pi talk to the drive over that ribbon cable, plus the assembly screws.

![Raspberry Pi 5 with heatsink, NVMe drive, and adapter board laid out for assembly](/img/rpi_bootstrapping/pi-components.jpeg)

Here are the other two raspberry pi nodes with the remaining parts.

![Raspberry Pi 5 with other parts boxed](/img/rpi_bootstrapping/pi-all.jpeg)

And here's the finished result, the Pi buttoned up inside its metal enclosure where it'll actually live alongside the other two raspberry pi 5s.

![Raspberry Pi 5 in its metal enclosure](/img/rpi_bootstrapping/pi-enclosure.jpeg)

## The Network: An Isolated VLAN

The networking is where I spent the most time, and it's also where I made the mistake that cost me the most to diagnose, so it's worth slowing down here. The cluster lives on its own isolated VLAN 200 that I created on the Cloud Gateway Fiber, with the gateway at `192.168.200.1` and no DHCP server running on it by design. Isolated means exactly that: my other home networks can't reach it, and the only way into the VLAN is for a device to have an interface that explicitly carries it.

{{< mermaid >}}
graph TB
    UCG["UCG-Fiber<br/>192.168.200.1<br/>VLAN 200 gateway"]

    subgraph HOST["Host Machine"]
        NIC["enp6s0f0<br/>bare NIC, no IP"]
        SUB["enp6s0f0.200<br/>192.168.200.2/24"]
        NIC -.- SUB
    end

    SW[("US-24 Switch")]

    subgraph CLUSTER["VLAN 200 (192.168.200.0/24)"]
        RPI1["rpi-node-01<br/>eth0: 192.168.200.10"]
        RPI2["rpi-node-02<br/>eth0: 192.168.200.11"]
        RPI3["rpi-node-03<br/>eth0: 192.168.200.12"]
    end

    UCG --- SW
    SUB -->|"Lab-Pi-Trunk<br/>802.1Q tagged VLAN 200"| SW
    SW -->|"Lab-Pi-Access<br/>VLAN 200 untagged"| RPI1
    SW -->|"Lab-Pi-Access<br/>VLAN 200 untagged"| RPI2
    SW -->|"Lab-Pi-Access<br/>VLAN 200 untagged"| RPI3
{{< /mermaid >}}

Getting devices onto that network takes two different port profiles on the switch, and this is the part that tripped me up. The Pis and my desktop have opposite tagging needs. The Pis use an access profile I called `Lab-Pi-Access`, where the native VLAN is 200 and everything else is blocked, so the nodes receive untagged frames on `eth0` and never see a VLAN tag at all. My desktop uses a trunk profile, `Lab-Pi-Trunk`, where the native VLAN stays the default and VLAN 200 is tagged, so the host can send 802.1Q-tagged frames into the cluster network. The mistake I made early on was applying the incorrect profile to a Pi port, which caused the node to simply go dark. Tagged frames arrive, `eth0` expects untagged, and the node is completely unreachable with nothing obvious to point at. I also managed, on the very first Pi, to plug it into the entirely wrong switch port, which is exactly the kind of thing that has you facepalming once you finally spot it. After unplugging it, moving it to the right port, and reflashing the drive so cloud-init would run again, it came up fine. Between the wrong port and the trunk-versus-access confusion, I leaned on my AI assistant to work through what was actually happening before it clicked.

## Giving the Host Access

So how does my desktop, sitting on the default LAN, reach an isolated VLAN with no DHCP? It uses one of the ports on that quad-port card with a VLAN 200 subinterface. The parent NIC carries no IP of its own, and all the cluster traffic flows through the tagged subinterface instead.

```yaml
vlans:
  enp6s0f0.200:
    id: 200
    link: enp6s0f0
    dhcp4: false
    dhcp6: false
    addresses:
      - 192.168.200.2/24
```

The subinterface adds an 802.1Q tag to every outgoing frame, which matches what the trunk port on the switch expects. Without that tag the switch treats the traffic as the default VLAN and the Pis stay unreachable, which is the host-side mirror of the same tagging problem I hit on the switch. Once both profiles were in place, the host came alive on the VLAN, and I could ping the gateway at `192.168.200.1` and each node in turn. The realization that I needed two distinct port profiles, one for the dedicated cluster space and one to bridge my desktop into it, was the thing I didn't know going in, and it's what unblocked the whole network once I sorted it out.

## Verifying the Build

With networking sorted, the payoff is being able to SSH into all three nodes. The `admin` user is the only way in, since root SSH is disabled and password login isn't allowed, so it's strictly key-based. On each node I check `cloud-init status --long` to confirm it finished, and the actual record of what happened lives in `/var/log/cloud-init.log` and `/var/log/cloud-init-output.log` if I want to see the first-boot run in detail. Cloud-init did everything I asked, which honestly wasn't a huge amount of work once the image was doing the heavy lifting, and the only thing it complains about is a harmless warning I don't worry about. All three nodes are bootstrapped, reachable, and reproducible, which is exactly where I wanted to land before going anywhere near Kubernetes.

## Lessons Learned

A few things from this build are worth calling out, and most of them are Trixie-specific or homelab-specific gotchas rather than anything in a tutorial. The VLAN port-profile mismatch was the single hardest issue to diagnose, because the host and Pi ports look almost identical in the UniFi UI but need opposite native VLAN settings, and applying the trunk profile to a Pi port renders it completely unreachable with no helpful signal. The cloud-init config itself went through several iterations, since each node was bootstrapped with a slightly cleaner version than the last as I refined the process, which is also why the third node has the tidiest config of the three. Swap on Trixie surprised me too, because the Pi uses a zram-based generator rather than a swapfile, so a plain `swapoff -a` doesn't actually touch it and you have to disable the zram setup directly. The NTP ordering was its own small saga, where the missing real-time clock means you genuinely cannot run apt until you've forced a time sync first, and skipping that step just fails in a confusing way. Networking on Trixie also runs through NetworkManager rather than Netplan, so the static IPs come from the cloud-init `network-config` file applied by NetworkManager rather than from editing Netplan files the way I would on Ubuntu, and I confirmed that the hard way by trying to stuff the network config into `user-data` first and watching it not work. The last thing isn't really a problem so much as a design choice that paid off, which is keeping the image-level work in `bake.sh` separate from the per-node work, because that separation is what made reprovisioning a node fast and unattended.

Next up is the part this whole exercise was building toward: actually deploying Kubernetes onto these three nodes with kubeadm so I can keep practicing ahead of the CKA. I also want to see whether I can stand up a local cache for images and packages inside this VLAN, since I already figured that out for my virtual machines but haven't done it for real hardware yet. Those are both their own posts, but the cluster is finally ready for them.

Well, that's all. See you in the next one.

**P.S.** I used Claude Code as a sounding board while writing this post, and full disclosure, I leaned on an AI assistant during the build itself to work through the NTP clock issue and the VLAN port-profile confusion. The ideas, the troubleshooting, and the final edits are my own.
