# NVIDIA BlueField DPU Network Configuration Documentation

## Network Card Driver Installation

### If No Ethernet: Ethernet Driver Setup

1. Download the appropriate Realtek ethernet driver from the [official Realtek downloads page](https://www.realtek.com/Download/List?cate_id=584)
2. Copy the driver to a USB drive
3. Install the driver on the target machine by running the `autorun.sh` script included in the downloaded package

---

## DOCA and MST Installation on Ubuntu Host

1. Follow the [official DOCA host installation guide](https://docs.nvidia.com/doca/sdk/doca-host+installation+and+upgrade/index.html)
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

1. Follow the [BlueField Bundle Installation Guide](https://docs.nvidia.com/doca/sdk/bf-bundle-installation-and-upgrade/index.html)
2. Download the correct BlueField installation package using the following settings:
   - **Deployment Platform:** BlueField
   - **Deployment Package:** BF-Bundle
   - **Distribution:** Ubuntu 24.04_64k
   - **Installer Type:** BFB
   - [Direct download link](https://developer.nvidia.com/doca-downloads?deployment_platform=BlueField&deployment_package=BF-Bundle&Distribution=Ubuntu&version=24.04_64k&installer_type=BFB)
3. Install the `.bfb` file onto the BlueField DPU:
   - Choose **"No pre-defined password"** option unless you want to set a specific password
   - **Note:** The ISO file is not required

### Troubleshooting: rshim Not Found

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

MST (Mellanox Software Tools) does not start automatically on reboot. Without MST running, you will not be able to manage or communicate with the DPU from the host. You need to create a systemd service on **both host machines** to ensure MST starts on boot.

### Step 1: Verify the MST Binary Path

Before creating the service, confirm where the MST binary is installed:

```bash
which mst
```

This will return the full path to the binary. It is typically located at `/usr/bin/mst`. If your path differs, update the `ExecStart`, `ExecStop`, and `ExecRestart` lines in the service file below accordingly.

### Step 2: Create the Service File

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

> **Reminder:** Repeat these steps on **both host machines**.

---

## Disabling Open vSwitch

Open vSwitch can create bridge interfaces that interfere with proper communication between compute nodes. Disable it on both hosts to avoid connectivity issues:

```bash
sudo systemctl stop openvswitch-switch
sudo systemctl disable openvswitch-switch
```

> **Note:** This must be done on **both hosts**.

---

## Machine Network Configuration

### Laka Machine

**Hostname:** `laka.medianet.cs.kent.edu`

**Ethernet Interface (enp7s0):**
- IP Address: `10.37.11.91/24`
- DNS: `10.32.4.38, 10.32.4.39`

### Hina Machine

**Hostname:** `hina.medianet.cs.kent.edu`

**Ethernet Interface (enp7s0):**
- IP Address: `10.37.11.92/24`
- DNS: `10.32.4.38, 10.32.4.39`

### Host-to-DPU Communication IP Addressing

All Host-to-DPU communication uses the `192.168.100.x` subnet.

> **Recommended:** Use the netplan configuration files in the Reference section below for automatic IP address and routing setup.

#### Laka IP Assignments
- **Laka Host (enp1s0f1np1):** `192.168.100.10`
- **Laka PCIe (Host-to-DPU):** `192.168.100.11`
- **Laka DPU P1 (DPU-to-DPU):** `192.168.5.11`
- **Laka DPU P0:** `192.168.6.11`

#### Hina IP Assignments
- **Hina Host (enp1s0f1np1):** `192.168.100.20`
- **Hina PCIe (Host-to-DPU):** `192.168.100.12`
- **Hina DPU P1 (DPU-to-DPU):** `192.168.5.12`
- **Hina DPU P0:** `192.168.6.12`

---

## DPU Configuration

### Accessing the DPU Controller Interface

The DPU exposes a virtual network device on the host called `tmfifo_net0` (or similar) that allows you to SSH directly into the DPU's ARM-based operating system for initial configuration.

#### Step 1: Find the tmfifo_net Interface Name

The interface name may vary between machines and kernel versions. To find it, run:

```bash
ip link show | grep tmfifo
```

Or list all network interfaces and look for one matching the `tmfifo_net` pattern:

```bash
ls /sys/class/net/ | grep tmfifo
```

On a **single-DPU system**, you will typically see:
- `tmfifo_net0`

On a **multi-DPU system**, each DPU gets its own tmfifo interface:
- First DPU: `tmfifo_net0`
- Second DPU: `tmfifo_net1`
- And so on...

Make note of the correct interface name before proceeding.

#### Step 2: Assign an IP to the tmfifo_net Interface

Replace `tmfifo_net0` with the correct interface name for your DPU if it differs:

```bash
sudo ip addr add 192.168.100.1/30 dev tmfifo_net0
```

> **Multi-DPU Note:** Each tmfifo interface needs a **separate, non-overlapping subnet**. For example:
> - First DPU: `192.168.100.1/30` (DPU reachable at `192.168.100.2`)
> - Second DPU: `192.168.100.5/30` (DPU reachable at `192.168.100.6`)

#### Step 3: SSH into the DPU

```bash
ssh ubuntu@192.168.100.2
```

- Default password: `ubuntu`
- **Note:** Upon first login, you will be forced to set a new password (minimum 12 characters)
- Current password: Set to the standard research machine password (entered twice during setup)

#### Step 4: Flush the tmfifo_net Interface When Done

After you have finished configuring the DPU, flush the IP from the tmfifo interface, as it will interfere with the persistent IP addressing configured via netplan:

```bash
sudo ip addr flush dev tmfifo_net0
```

---

### Enabling IPv4 Routing on DPUs

Enable IP forwarding on both DPUs:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

To make this persistent across reboots, add the following line to `/etc/sysctl.conf`:

```
net.ipv4.ip_forward=1
```

---

## Reference: Netplan Configuration Files

> **Note 1:** After adding each file to the correct folder, run `sudo netplan apply` to activate the configuration.

> **Note 2:** You may need to run `sudo chmod 600 <name_of_configuration_file>` to suppress permission warning messages.

### Hina Host Configuration

**File:** `/etc/netplan/01-network-manager-all.yaml`

```yaml
# Let NetworkManager manage all devices on this system
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
        addresses: [10.32.4.28, 10.32.4.39]
    enp1s0f1np1:
      addresses:
        - 192.168.100.20/24
      routes:
        - to: 192.168.100.10/32
          via: 192.168.100.12
```

### Hina DPU Configuration

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    pf1hpf:
      addresses:
        - 192.168.100.12/24
    p1:
      addresses:
        - 192.168.5.12/24
      routes:
        - to: 192.168.100.10/32
          via: 192.168.5.11
    p0:
      addresses:
        - 192.168.6.12/24
```

### Laka Host Configuration

**File:** `/etc/netplan/01-network-manager-all.yaml`

```yaml
# Let NetworkManager manage all devices on this system
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
    enp1s0f1np1:
      addresses:
        - 192.168.100.10/24
      routes:
        - to: 192.168.100.20/32
          via: 192.168.100.11
```

### Laka DPU Configuration

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    pf1hpf:
      addresses:
        - 192.168.100.11/24
    p1:
      optional: true
      addresses:
        - 192.168.5.11/24
      routes:
        - to: 192.168.100.20/32
          via: 192.168.5.12
    p0:
      optional: true
      addresses:
        - 192.168.6.11/24
```

---

## Additional Resources

- [NVIDIA BlueField DPU Host-Side Interface Configuration](https://docs.nvidia.com/networking/display/bluefielldpuosv452/host-side%2Binterface%2Bconfiguration)
- [NVIDIA DOCA SDK Documentation](https://docs.nvidia.com/doca/)
- [NVIDIA Developer Portal](https://developer.nvidia.com/)
