# NVIDIA BlueField DPU Network Configuration Documentation

## Network Card Driver Installation

### If No Ethernet: Ethernet Driver Setup

1. Download the appropriate Realtek ethernet driver from the [official Realtek downloads page](https://www.realtek.com/Download/List?cate_id=584)
2. Copy the driver to a USB drive
3. Install the driver on the target machine by running the `autorun.sh` script included in the downloaded package

---
## NVIDIA GPU Driver Installation

### When GPU Drivers Are Also Needed

When a machine requires both DPU and GPU drivers, the GPU driver must be installed manually with specific flags to avoid conflicts. Do not use package managers or other tools to install the driver.

The default NVIDIA GPU driver installation includes components that interfere with the DPU:

- **`--no-peermem`** — Disables `nvidia-peermem`, which enables direct peer-to-peer read/write access to GPU video memory. This conflicts with the DPU’s own peermem module.
- **`--no-dkms`** — Disables Dynamic Kernel Module Support (DKMS), which automatically rebuilds the driver on kernel updates. This may require manual reinstallation after kernel upgrades, but ensures the driver remains compatible with the DPUs.
- **`--no-nouveau-check`** — Skips the check for the open-source Nouveau driver.
- **`--no-kernel-module-source`** — Skips installation of the kernel module source files.
- **`--no-rebuild-initramfs`** — Skips rebuilding the initial RAM filesystem.

### Installation Steps

1. Download the appropriate driver from NVIDIA. For GeForce RTX 5080 (driver version 570): [NVIDIA Driver Download](https://www.nvidia.com/en-us/drivers/details/261111/)
2. Run the installer with the required flags:

```bash
sudo sh ./NVIDIA-Linux-x86_64-570.211.01.run --no-peermem --no-dkms --no-nouveau-check --no-kernel-module-source --no-rebuild-initramfs
```

---

## DOCA and MST Installation on Ubuntu Host
> **Note:** This guide is for ubuntu 22.04. It may change based on using Ubuntu 24 or Ubuntu 26.

> **Machine:** All host machines (Laka, Hina, and Milu)

1. Follow the [official DOCA host installation guide](https://docs.nvidia.com/doca/sdk/DOCA-Host-Installation-and-Upgrade/index.html)
2. Download the correct DOCA package for the host using the following settings:
   - **Deployment Platform:** Host-Server
   - **Deployment Package:** DOCA-Host
   - **Target OS:** Linux (x86_64)
   - **Profile:** doca-all
   - **Distribution:** Ubuntu 22.04
   - **Installer Type:** deb_local
   - [Direct download link](https://developer.nvidia.com/doca-downloads?deployment_platform=Host-Server&deployment_package=DOCA-Host&target_os=Linux&Architecture=x86_64&Profile=doca-all&Distribution=Ubuntu&version=22.04&installer_type=deb_local)
3. **Important:** Follow all steps in the guide to load the driver and initialize MST

---

## BlueField DPU Firmware Installation

> **Machine:** All host machines (Laka, Hina, and Milu)

1. Follow the [BlueField Bundle Installation Guide](https://docs.nvidia.com/doca/sdk/BF-Bundle-Installation-and-Upgrade/index.html)
2. Download the correct BlueField installation package using the following settings:
   - **Deployment Platform:** BlueField
   - **Deployment Package:** BF-Bundle
   - **Distribution:** Ubuntu 24.04_64k
   - **Installer Type:** BFB
   - [Direct download link](https://developer.nvidia.com/doca-downloads?deployment_platform=BlueField&deployment_package=BF-Bundle&Distribution=Ubuntu&version=24.04_64k&installer_type=BFB)
3. Install the `.bfb` file onto each BlueField DPU:
   - Choose **"No pre-defined password"** option unless you want to set a specific password
   - **Note:** The ISO file is not required

### Troubleshooting: rshim Not Found

> **Machine:** All host machines (Laka, Hina, and Milu)

If the `rshim` device is not detected during installation, start and enable the rshim service manually:

```bash
sudo systemctl start rshim
sudo systemctl enable rshim
```

#### Multi-DPU Systems: rshim Numbering

If your machine has **more than one DPU**, each DPU will be assigned its own rshim device. These are numbered starting at 0:

- First DPU: `/dev/rshim0/`
- Second DPU: `/dev/rshim1/`
- And so on...

You can list all detected rshim devices with:

```bash
ls /dev/ | grep rshim
```

When flashing a specific DPU, make sure to target the correct rshim device. For example, to flash the second DPU:

```bash
sudo bfb-install --rshim rshim1 --bfb <path-to-your.bfb>
```

---

## MST Systemd Service Configuration

> **Machine:** All host machines (Laka, Hina, and Milu)

MST (Mellanox Software Tools) does not start automatically on reboot. Without MST running, you will not be able to manage or communicate with the DPU from the host. You need to create a systemd service on **all host machines** to ensure MST starts on boot.

### Step 1: Verify the MST Binary Path

> **Machine:** All host machines (Laka, Hina, and Milu)

Before creating the service, confirm where the MST binary is installed:

```bash
which mst
```

This will return the full path to the binary. It is typically located at `/usr/bin/mst`. If your path differs, update the `ExecStart`, `ExecStop`, and `ExecRestart` lines in the service file below accordingly.

### Step 2: Create the Service File

> **Machine:** All host machines (Laka, Hina, and Milu)

Create the systemd service file at `/etc/systemd/system/mst.service`:

```bash
sudo nano /etc/systemd/system/mst.service
```

Paste in the following content:

```ini
[Unit]
Description=Mellanox MST Initialization
After=systemd-modules-load.service pci-bus.target
Requires=systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/usr/bin/mst start
ExecStop=/usr/bin/mst stop
ExecRestart=/usr/bin/mst restart
RemainAfterExit=yes
TimeoutSec=60

[Install]
WantedBy=multi-user.target
```

**How this service works:**
- `Type=oneshot` — systemd runs the command once and considers the service "done" after it exits. This is appropriate for initialization commands like `mst start`.
- `RemainAfterExit=yes` — even though the process exits after running, systemd still considers the service "active." This allows `systemctl stop` to later call `ExecStop`.
- `ExecStart` / `ExecStop` / `ExecRestart` — these map directly to the `mst start`, `mst stop`, and `mst restart` commands you would otherwise run manually.
- `After` and `Requires` — ensures the service waits for kernel modules to be loaded before attempting to start MST.

### Step 3: Enable and Start the Service

> **Machine:** All host machines (Laka, Hina, and Milu)

After saving the file, reload systemd and enable the service so it starts automatically on every boot:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mst.service
sudo systemctl start mst.service
```

Verify the service is running correctly:

```bash
sudo systemctl status mst.service
```

> **Reminder:** Repeat these steps on **all host machines**.

---

## DPU Configuration for Single-DPU Systems

> **Machine:** Laka and Hina host machines

The DPU exposes a virtual network device on the host called `tmfifo_net0` that allows you to SSH directly into the DPU's ARM-based operating system for initial configuration.

### Step 1: Configure the tmfifo Interface on the Host

Create or edit a netplan configuration file on the host to assign an IP to the `tmfifo_net0` interface:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    tmfifo_net0:
      dhcp4: false
      addresses:
        - "192.168.100.1/30"
```

Apply the configuration:

```bash
sudo netplan apply
```

### Step 2: SSH into the DPU

```bash
ssh ubuntu@192.168.100.2
```

- Default credentials: user `ubuntu`, password `ubuntu`
- Upon first login, you will be forced to set a new password (minimum 12 characters)

---

## DPU Configuration for Multi-DPU Systems

> **Note:** Specifically for the Milu system only

### Accessing the DPU Controller Interface

The DPU exposes a virtual network device on the host called `tmfifo_net0` (or similar) that allows you to SSH directly into the DPU's ARM-based operating system for initial configuration.

On systems with **multiple DPUs**, each card is configured with its own unique tmfifo IP address by editing the netplan configuration inside the card via the rshim console. This avoids routing conflicts that would occur if all DPUs shared the same address.

#### Step 1: Open the rshim Console for the Target DPU

> **Machine:** Milu machine

Connect to the DPU's serial console through its rshim device. For DPU 1 (rshim0):

```bash
sudo screen /dev/rshim0/console
```

For DPU 2 (rshim1):

```bash
sudo screen /dev/rshim1/console
```

Log in with the DPU's credentials when prompted.

Default credentials are...
  - user: `ubuntu`
  - password: `ubuntu`

#### Step 2: Configure a Unique tmfifo IP on the DPU

> **Machine:** Inside the DPU (via rshim console)

Each DPU must be assigned a unique tmfifo IP so that multiple DPUs can be accessed simultaneously without address conflicts. Edit the cloud-init netplan configuration file on the DPU:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Set the addresses according to which DPU you are configuring:

- **DPU 1 (rshim0):** use `192.168.101.1/30` on `tmfifo_net0`
- **DPU 2 (rshim1):** use `192.168.102.1/30` on `tmfifo_net1`

**Example configuration for DPU 1 machine:**

```yaml
network:
  version: 2
  ethernets:
    oob_net0:
      renderer: NetworkManager
      dhcp4: true
    tmfifo_net0:
      renderer: NetworkManager
      addresses:
      - "192.168.101.2/30"
      dhcp4: false
      routes:
      - metric: 1025
        to: "0.0.0.0/0"
        via: "192.168.101.1"
```

After saving the file, apply the configuration:

```bash
sudo netplan apply
```

Repeat this process (Steps 1–2) for each DPU, adjusting the subnet accordingly.

#### Step 3: SSH into the DPU

> **Machine:** Host machine (Laka, Hina, or Milu) — this command connects you to the DPU

Once the tmfifo IP has been configured on the DPU, you must set up the `tmfifo` interfaces on the host. Here is an example:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    tmfifo_net0:
      dhcp4: false
      addresses:
        - "192.168.101.1/30"
    tmfifo_net1:
      dhcp4: false
      addresses:
        - "192.168.102.1/30"
```
Then SSH into it using the corresponding address:

For DPU 1:
```bash
ssh ubuntu@192.168.101.2
```

For DPU 2:
```bash
ssh ubuntu@192.168.102.2
```

- Default password: `ubuntu`
- **Note:** Upon first login, you will be forced to set a new password (minimum 12 characters)
- Current password: Set to the standard research machine password

---

## Configuring Open vSwitch Bridges

> **Machine:** All DPU machines (requires SSH access — see DPU Configuration sections above)

Open vSwitch creates proper bridging between DPUs which helps to limit the load on the CPU. In order for the DPUs to properly work, you must set up two bridges.

One bridge is for the communication between the host and the DPU (`pf0hpf` is the interface for this), and the other bridge is for the communication between the two DPUs (`p0` is the interface for this).

Only one IP address can be assigned per bridge, so we need two separate bridges to have two separate IP addresses.

Running `ovs-vsctl show` will show you how the bridges are currently configured. For the purpose of this setup guide we will only worry about ports: `pf0hpf` and `p0`.

To properly configure these two ports on separate bridges, you need to first delete `p1` and `pf1hpf` via:
```bash
ovs-vsctl del-port ovsbr2 p1
ovs-vsctl del-port ovsbr2 pf1hpf
```

Once these two ports are deleted from their bridge, we can move `pf0hpf` to the `ovsbr2` bridge.
```bash
ovs-vsctl del-port ovsbr1 pf0hpf
ovs-vsctl add-port ovsbr2 pf0hpf
```

Here is an example of what a proper `ovs-vsctl show` looks like:

```
Bridge ovsbr2
  fail_mode: standalone
  Port pf0hpf
    Interface pf0hpf
  Port ovsbr2
    Interface ovsbr2
      type: internal
Bridge ovsbr1
  fail_mode: standalone
  Port p0
    Interface p0
  Port ovsbr1
    Interface ovsbr1
      type: internal
ovs_version: x
```

Once the bridges have been set up, you can move onto setting up the network across all the machines.

> **Note:** This bridge setup must be done on **all DPUs**.

---

## Machine Network Configuration

> **Note:** At the end of this MD file, you will find all the netplan configurations used for this guide.

### Subnet Overview

Laka DPU and Milu DPU1 share the `192.168.3.0/24` subnet for DPU-to-DPU communication (ovsbr1). Hina DPU and Milu DPU2 share the `192.168.4.0/24` subnet for DPU-to-DPU communication (ovsbr1). Milu sits in the middle of both, acting as the bridge between the two sides. Each host-to-DPU link (ovsbr2) uses its own subnet:

| Subnet | Purpose |
|--------|---------|
| `192.168.3.0/24` | DPU-to-DPU: Laka DPU ↔ Milu DPU1 (ovsbr1) |
| `192.168.4.0/24` | DPU-to-DPU: Hina DPU ↔ Milu DPU2 (ovsbr1) |
| `192.168.5.0/24` | Host-to-DPU: Laka Host ↔ Laka DPU (ovsbr2) |
| `192.168.6.0/24` | Host-to-DPU: Milu Host ↔ Milu DPU2 (ovsbr2) |
| `192.168.7.0/24` | Host-to-DPU: Milu Host ↔ Milu DPU1 (ovsbr2) |
| `192.168.8.0/24` | Host-to-DPU: Hina Host ↔ Hina DPU (ovsbr2) |

### Laka Machine

**Hostname:** `laka.medianet.cs.kent.edu`

| Interface | IP Address | Purpose |
|-----------|-----------|---------|
| enp7s0 | `10.37.11.91/24` | Management (DNS: `10.32.4.38, 10.32.4.39`) |
| enp1s0f0np0 | `192.168.5.1/24` | Host-to-DPU |
| DPU ovsbr2 (pf0hpf) | `192.168.5.2/24` | Host-to-DPU (DPU side) |
| DPU ovsbr1 (p0) | `192.168.3.2/24` | DPU-to-DPU (→ Milu DPU1) |

### Hina Machine

**Hostname:** `hina.medianet.cs.kent.edu`

| Interface | IP Address | Purpose |
|-----------|-----------|---------|
| enp7s0 | `10.37.11.92/24` | Management (DNS: `10.32.4.38, 10.32.4.39`) |
| enp1s0f0np0 | `192.168.8.1/24` | Host-to-DPU |
| DPU ovsbr2 (pf0hpf) | `192.168.8.2/24` | Host-to-DPU (DPU side) |
| DPU ovsbr1 (p0) | `192.168.4.2/24` | DPU-to-DPU (→ Milu DPU2) |

### Milu Machine

**Hostname:** `milu.medianet.cs.kent.edu`

> **Note:** Milu hosts multiple DPUs. Refer to the Multi-DPU sections throughout this document for rshim numbering, tmfifo interface naming, and subnet assignments when working with Milu.

| Interface | IP Address | Purpose |
|-----------|-----------|---------|
| eno2np1 | `10.37.11.90/24` | Management (DNS: `10.32.4.38, 10.32.4.39`) |
| enp1s0f0np0 | `192.168.7.1/24` | Host-to-DPU1 |
| enp193s0f0np0 | `192.168.6.1/24` | Host-to-DPU2 |
| DPU1 ovsbr2 (pf0hpf) | `192.168.7.2/24` | Host-to-DPU (DPU1 side) |
| DPU1 ovsbr1 (p0) | `192.168.3.1/24` | DPU-to-DPU (→ Laka DPU) |
| DPU2 ovsbr2 (pf0hpf) | `192.168.6.2/24` | Host-to-DPU (DPU2 side) |
| DPU2 ovsbr1 (p0) | `192.168.4.1/24` | DPU-to-DPU (→ Hina DPU) |

---

# Building the Network

Once all the cards and DPUs have been set up, it is time to set up the network via netplan.

## Enable IP Forwarding

> **Machine:** All DPUs and Milu Host

Every device that forwards traffic between subnets must have IP forwarding enabled. Without this, packets will arrive at a hop but be dropped instead of forwarded to the next subnet.

Enable it immediately:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

To make it persistent across reboots, add or uncomment the following line in `/etc/sysctl.conf`:

```
net.ipv4.ip_forward=1
```

Then apply:

```bash
sudo sysctl -p
```

> **Note:** This must be done on **all DPUs** (Hina, Laka, Milu DPU1, Milu DPU2) and on **Milu Host**, since they all act as forwarding hops in the network path.

---

## Reference: Netplan Configuration Files

**Note 1:** After adding each file to the correct folder, run `sudo netplan apply` to activate the configuration.

**Note 2:** You may need to run `sudo chmod 600 <name_of_configuration_file>` to suppress permission warning messages.

**Note 3: Bidirectional Routing** — The routes configured below support traffic in both directions. Hina can reach Laka and Laka can reach Hina. The full paths are:

**Hina → Laka:**
`Hina Host → Hina DPU → Milu DPU2 → Milu Host → Milu DPU1 → Laka DPU → Laka Host`

**Laka → Hina:**
`Laka Host → Laka DPU → Milu DPU1 → Milu Host → Milu DPU2 → Hina DPU → Hina Host`

### Hina Host Configuration

**Machine:** Hina host machine

**File:** `/etc/netplan/01-network-manager-all.yaml`

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp7s0:
      dhcp4: no
      addresses:
        - 10.37.11.92/24
      routes:
        - to: default
          via: 10.37.11.1
      nameservers:
        addresses: [10.32.4.38, 10.32.4.39]

    enp1s0f0np0:
      addresses:
        - 192.168.8.1/24
      routes:
        - to: 192.168.4.0/24
          via: 192.168.8.2
        - to: 192.168.6.0/24
          via: 192.168.8.2
        - to: 192.168.5.0/24
          via: 192.168.8.2
        - to: 192.168.7.0/24
          via: 192.168.8.2
        - to: 192.168.3.0/24
          via: 192.168.8.2
```

### Hina DPU Configuration

> **Machine:** Hina DPU (via SSH from Hina host)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    p0:
      dhcp4: false
    pf0hpf:
      dhcp4: false
  bridges:
    ovsbr1:
      interfaces: [p0]
      openvswitch: {}
      addresses:
        - 192.168.4.2/24
      optional: true
      routes:
        - to: 192.168.6.0/24
          via: 192.168.4.1
        - to: 192.168.7.0/24
          via: 192.168.4.1 
        - to: 192.168.5.0/24
          via: 192.168.4.1  
        - to: 192.168.3.0/24
          via: 192.168.4.1 
    ovsbr2:
      interfaces: [pf0hpf]
      openvswitch: {}
      addresses:
        - 192.168.8.2/24

```

### Laka Host Configuration

> **Machine:** Laka host machine

**File:** `/etc/netplan/01-network-manager-all.yaml`

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp7s0:
      dhcp4: no
      addresses:
        - 10.37.11.91/24
      routes:
        - to: default
          via: 10.37.11.1
      nameservers:
        addresses: [10.32.4.38, 10.32.4.39]

    enp1s0f0np0:
      addresses:
        - 192.168.5.1/24
      routes:
        - to: 192.168.3.0/24
          via: 192.168.5.2
        - to: 192.168.7.0/24
          via: 192.168.5.2
        - to: 192.168.8.0/24
          via: 192.168.5.2
        - to: 192.168.6.0/24
          via: 192.168.5.2
        - to: 192.168.4.0/24
          via: 192.168.5.2

```

### Laka DPU Configuration

> **Machine:** Laka DPU (via SSH from Laka host)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    p0:
      dhcp4: false
    pf0hpf:
      dhcp4: false
  bridges:
    ovsbr1:
      interfaces: [p0]
      openvswitch: {}
      addresses:
        - 192.168.3.2/24
      optional: true
      routes:
        - to: 192.168.7.0/24
          via: 192.168.3.1
        - to: 192.168.8.0/24
          via: 192.168.3.1
        - to: 192.168.4.0/24
          via: 192.168.3.1
        - to: 192.168.6.0/24
          via: 192.168.3.1
    ovsbr2:
      interfaces: [pf0hpf]
      openvswitch: {}
      addresses:
        - 192.168.5.2/24


```

### Milu Host Configuration

> **Machine:** Milu host machine

**File:** `/etc/netplan/01-network-manager-all.yaml`

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eno2np1:
      dhcp4: no
      addresses:
        - 10.37.11.90/24
      routes:
        - to: default
          via: 10.37.11.1
      nameservers:
        addresses: [10.32.4.38, 10.32.4.39]

    enp1s0f0np0:
      addresses:
        - 192.168.7.1/24
      routes:
        - to: 192.168.5.0/24
          via: 192.168.7.2
        - to: 192.168.3.0/24
          via: 192.168.7.2

    enp193s0f0np0:
      addresses:
        - 192.168.6.1/24
      routes:
        - to: 192.168.4.0/24
          via: 192.168.6.2
        - to: 192.168.8.0/24
          via: 192.168.6.2
```

### Milu DPU 1 Configuration

> **Machine:** Milu DPU 1 (via SSH from Milu host using tmfifo_net0)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    p0:
      dhcp4: false
    pf0hpf:
      dhcp4: false
  bridges:
    ovsbr1:
      interfaces: [p0]
      openvswitch: {}
      addresses:
        - 192.168.3.1/24
      optional: true
      routes:
        - to: 192.168.5.0/24
          via: 192.168.3.2
    ovsbr2:
      interfaces: [pf0hpf]
      openvswitch: {}
      addresses:
        - 192.168.7.2/24
      routes:
        - to: 192.168.8.0/24
          via: 192.168.7.1
        - to: 192.168.6.0/24
          via: 192.168.7.1

```

### Milu DPU 2 Configuration

> **Machine:** Milu DPU 2 (via SSH from Milu host using tmfifo_net1)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    p0:
      dhcp4: false
    pf0hpf:
      dhcp4: false
  bridges:
    ovsbr1:
      interfaces: [p0]
      openvswitch: {}
      addresses:
        - 192.168.4.1/24
      optional: true
      routes:
        - to: 192.168.8.0/24
          via: 192.168.4.2
    ovsbr2:
      interfaces: [pf0hpf]
      openvswitch: {}
      addresses:
        - 192.168.6.2/24
      routes:
        - to: 192.168.7.0/24
          via: 192.168.6.1
        - to: 192.168.5.0/24
          via: 192.168.6.1
```

---

## Additional Resources

- [NVIDIA BlueField DPU Host-Side Interface Configuration](https://docs.nvidia.com/networking/display/bluefieldbsp4140/Host-side-Interface-Configuration)
- [NVIDIA DOCA SDK Documentation](https://docs.nvidia.com/doca/sdk/index.html)
- [NVIDIA Developer Portal](https://developer.nvidia.com/)
