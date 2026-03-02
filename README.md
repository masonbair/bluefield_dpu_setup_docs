# NVIDIA BlueField DPU Network Configuration Documentation

## Network Card Driver Installation

### If No Ethernet: Ethernet Driver Setup

1. Download the appropriate Realtek ethernet driver from the [official Realtek downloads page](https://www.realtek.com/Download/List?cate_id=584)
2. Copy the driver to a USB drive
3. Install the driver on the target machine by running the `autorun.sh` script included in the downloaded package

---

## DOCA and MST Installation on Ubuntu Host

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

## Disabling Open vSwitch

> **Machine:** All host machines (Laka, Hina, and Milu)

Open vSwitch can create bridge interfaces that interfere with proper communication between compute nodes. Disable it on all hosts to avoid connectivity issues:

```bash
sudo systemctl stop openvswitch-switch
sudo systemctl disable openvswitch-switch
```

> **Note:** This must be done on **all hosts**.

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

### Milu Machine

**Hostname:** `milu.medianet.cs.kent.edu`

> **Note:** Milu hosts multiple DPUs. Refer to the Multi-DPU sections throughout this document for rshim numbering, tmfifo interface naming, and subnet assignments when working with Milu.

**Ethernet Interface (enp7s0):**
- IP Address: `10.37.11.90/24`
- DNS: `10.32.4.38, 10.32.4.39`

---

### Host-to-DPU Communication IP Addressing

Hina and Milu DPU1 share the `192.168.10.0/28` subnet. Laka and Milu DPU2 share the `192.168.20.0/28` subnet. Milu sits in the middle of both, acting as the bridge between the two sides.

> **Recommended:** Use the netplan configuration files in the Reference section below for automatic IP address and routing setup.

#### Hina IP Assignments
- **Hina Host:** `192.168.10.7/28`
- **Hina PCIe (Host-to-DPU):** `192.168.10.6`
- **Hina DPU P0 (DPU-to-DPU):** `192.168.10.4`
- **Hina DPU P1 (DPU-to-DPU):** `192.168.10.5`

#### Laka IP Assignments
- **Laka Host:** `192.168.20.7/28`
- **Laka PCIe (Host-to-DPU):** `192.168.20.6`
- **Laka DPU P0 (DPU-to-DPU):** `192.168.20.4`
- **Laka DPU P1 (DPU-to-DPU):** `192.168.20.5`

#### Milu IP Assignments

Milu has two host interfaces, one per DPU, each on a separate subnet:

**DPU 1 — subnet `192.168.10.0/28` (shared with Hina)**
- **Milu Host Interface (enp2):** `192.168.10.0/28`
- **Milu PCIe 1 (Host-to-DPU):** `192.168.10.1`
- **Milu DPU 1 P0 (DPU-to-DPU):** `192.168.10.2`
- **Milu DPU 1 P1 (DPU-to-DPU):** `192.168.10.3`

**DPU 2 — subnet `192.168.20.0/28` (shared with Laka)**
- **Milu Host Interface (enp255):** `192.168.20.0/28`
- **Milu PCIe 2 (Host-to-DPU):** `192.168.20.1`
- **Milu DPU 2 P0 (DPU-to-DPU):** `192.168.20.2`
- **Milu DPU 2 P1 (DPU-to-DPU):** `192.168.20.3`

---

## DPU Configuration

### Accessing the DPU Controller Interface

The DPU exposes a virtual network device on the host called `tmfifo_net0` (or similar) that allows you to SSH directly into the DPU's ARM-based operating system for initial configuration.

> **Important:** The `192.168.100.1/30` address assigned to the host's tmfifo interface and the corresponding `192.168.100.2` address on the DPU side are **temporary addresses used exclusively for SSH access during initial DPU configuration.** These addresses must be flushed after configuration is complete, as they will conflict with the persistent IP addressing configured via netplan.

#### Step 1: Find the tmfifo_net Interface Name

> **Machine:** Host machine (Laka, Hina, or Milu)

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

> **Machine:** Host machine (Laka, Hina, or Milu)

Replace `tmfifo_net0` with the correct interface name for your DPU if it differs:

```bash
sudo ip addr add 192.168.100.1/30 dev tmfifo_net0
```

> **Multi-DPU Note:** Each tmfifo interface needs a **separate, non-overlapping subnet**. You must only have one tmfifo interface assigned an IP at a time — see the [Multi-DPU SSH Workflow](#multi-dpu-ssh-workflow) section below before proceeding if you are working with more than one DPU.

#### Step 3: SSH into the DPU

> **Machine:** Host machine (Laka, Hina, or Milu) — this command connects you to the DPU

```bash
ssh ubuntu@192.168.100.2
```

- Default password: `ubuntu`
- **Note:** Upon first login, you will be forced to set a new password (minimum 12 characters)
- Current password: Set to the standard research machine password (entered twice during setup)

#### Step 4: Flush the tmfifo_net Interface When Done

> **Machine:** Host machine (Laka, Hina, or Milu)

After you have finished configuring the DPU, flush the IP from the tmfifo interface, as it will interfere with the persistent IP addressing configured via netplan:

```bash
sudo ip addr flush dev tmfifo_net0
```

---

### Multi-DPU SSH Workflow

> **Machine:** Milu host machine (or any host with more than one DPU)

When a host has multiple DPUs, you **cannot** assign the same `192.168.100.1/30` subnet to more than one tmfifo interface at the same time — doing so will cause a routing conflict and SSH will connect to the wrong DPU. You must configure, use, and fully clean up each interface before moving on to the next.

Additionally, because both DPUs will respond at `192.168.100.2` on their respective subnets, SSH will store a host key for that address after your first login. When you switch to the second DPU, SSH will detect a key mismatch and refuse to connect. You must remove the old host key entry before SSHing into the next DPU.

#### Workflow for Each DPU

Repeat the following steps for each DPU, one at a time:

**1. Assign the IP to the tmfifo interface for the target DPU:**

For DPU 1:
```bash
sudo ip addr add 192.168.100.1/30 dev tmfifo_net0
```

For DPU 2:
```bash
sudo ip addr add 192.168.100.1/30 dev tmfifo_net1
```

**2. SSH into the DPU at `192.168.100.2`:**

```bash
ssh ubuntu@192.168.100.2
```

**3. Perform your configuration on the DPU, then exit the SSH session.**

**4. Flush the IP address from the tmfifo interface:**

For DPU 1:
```bash
sudo ip addr flush dev tmfifo_net0
```

For DPU 2:
```bash
sudo ip addr flush dev tmfifo_net1
```

**5. Remove the stored SSH host key for `192.168.100.2`:**

Because the next DPU will also present itself at `192.168.100.2`, the old host key will cause SSH to refuse the connection with a warning about a potential man-in-the-middle attack. Remove the stale key before proceeding:

```bash
ssh-keygen -R 192.168.100.2
```

**6. Repeat from step 1 for the next DPU.**

**Summary of the full sequence for Milu (two DPUs):**
```bash
# --- DPU 1 ---
sudo ip addr add 192.168.100.1/30 dev tmfifo_net0
ssh ubuntu@192.168.100.2          # configure DPU 1, then exit
sudo ip addr flush dev tmfifo_net0
ssh-keygen -R 192.168.100.2

# --- DPU 2 ---
sudo ip addr add 192.168.100.1/30 dev tmfifo_net1
ssh ubuntu@192.168.100.2          # configure DPU 2, then exit
sudo ip addr flush dev tmfifo_net1
ssh-keygen -R 192.168.100.2
````


---

## Reference: Netplan Configuration Files

**Note 1:** After adding each file to the correct folder, run `sudo netplan apply` to activate the configuration.

**Note 2:** You may need to run `sudo chmod 600 <name_of_configuration_file>` to suppress permission warning messages.

**Note 3: Bidirectional Routing** — The routes configured below support traffic in both directions. Hina can reach Laka and Laka can reach Hina. The full paths are:

**Hina → Laka:**
`Hina Host → Hina DPU → Milu DPU1 → Milu Host → Milu DPU2 → Laka DPU → Laka Host`

**Laka → Hina:**
`Laka Host → Laka DPU → Milu DPU2 → Milu Host → Milu DPU1 → Hina DPU → Hina Host`

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
    enp1s0f1np1:
      addresses:
        - 192.168.10.7/28
      routes:
        # Route to local DPU subnet and onward to the 192.168.20.0/28 (Laka) subnet via Hina DPU
        - to: 192.168.10.0/28
          via: 192.168.10.6
        - to: 192.168.20.0/28
          via: 192.168.10.6
```

### Hina DPU Configuration

> **Machine:** Hina DPU (via SSH from Hina host)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    pf1hpf:
      addresses:
        - 192.168.10.6/28
      routes:
        # Return route back to Hina host
        - to: 192.168.10.7/32
          via: 192.168.10.6
    p0:
      addresses:
        - 192.168.10.4/28
      routes:
        # Forward traffic toward Milu DPU1 and onward to the 192.168.20.0/28 (Laka) subnet
        - to: 192.168.10.0/28
          via: 192.168.10.2
        - to: 192.168.20.0/28
          via: 192.168.10.2
    p1:
      addresses:
        - 192.168.10.5/28
      routes:
        - to: 192.168.10.0/28
          via: 192.168.10.3
        - to: 192.168.20.0/28
          via: 192.168.10.3
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
    enp1s0f1np1:
      addresses:
        - 192.168.20.7/28
      routes:
        # Route to local DPU subnet and onward to the 192.168.10.0/28 (Hina) subnet via Laka DPU
        - to: 192.168.20.0/28
          via: 192.168.20.6
        - to: 192.168.10.0/28
          via: 192.168.20.6
```

### Laka DPU Configuration

> **Machine:** Laka DPU (via SSH from Laka host)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    pf1hpf:
      addresses:
        - 192.168.20.6/28
      routes:
        # Return route back to Laka host
        - to: 192.168.20.7/32
          via: 192.168.20.6
    p0:
      optional: true
      addresses:
        - 192.168.20.4/28
      routes:
        # Forward traffic toward Milu DPU2 and onward to the 192.168.10.0/28 (Hina) subnet
        - to: 192.168.20.0/28
          via: 192.168.20.2
        - to: 192.168.10.0/28
          via: 192.168.20.2
    p1:
      optional: true
      addresses:
        - 192.168.20.5/28
      routes:
        - to: 192.168.20.0/28
          via: 192.168.20.3
        - to: 192.168.10.0/28
          via: 192.168.20.3
```

### Milu Host Configuration

> **Machine:** Milu host machine

**File:** `/etc/netplan/01-network-manager-all.yaml`

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp7s0:
      dhcp4: no
      addresses:
        - 10.37.11.90/24
      routes:
        - to: default
          via: 10.37.11.1
      nameservers:
        addresses: [10.32.4.38, 10.32.4.39]
    enp2:
      addresses:
        - 192.168.10.0/28
      routes:
        # Reach Hina subnet via DPU1 and cross to 192.168.20.0/28 (Laka) via enp255
        - to: 192.168.10.0/28
          via: 192.168.10.1
        - to: 192.168.20.0/28
          via: 192.168.20.1
    enp255:
      addresses:
        - 192.168.20.0/28
      routes:
        # Reach Laka subnet via DPU2 and cross to 192.168.10.0/28 (Hina) via enp2
        - to: 192.168.20.0/28
          via: 192.168.20.1
        - to: 192.168.10.0/28
          via: 192.168.10.1
```

### Milu DPU 1 Configuration

> **Machine:** Milu DPU 1 (via SSH from Milu host using tmfifo_net0)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    pf1hpf:
      addresses:
        - 192.168.10.1/28
      routes:
        # Forward traffic from Hina side to Milu host, which bridges to DPU2 and onward to Laka
        - to: 192.168.20.0/28
          via: 192.168.10.1
    p0:
      optional: true
      addresses:
        - 192.168.10.2/28
      routes:
        # Route back toward Hina DPU and Hina host
        - to: 192.168.10.4/32
          via: 192.168.10.4
        - to: 192.168.10.7/32
          via: 192.168.10.4
    p1:
      optional: true
      addresses:
        - 192.168.10.3/28
      routes:
        - to: 192.168.10.5/32
          via: 192.168.10.5
        - to: 192.168.10.7/32
          via: 192.168.10.5
```

### Milu DPU 2 Configuration

> **Machine:** Milu DPU 2 (via SSH from Milu host using tmfifo_net1)

**File:** `/etc/netplan/99-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    pf1hpf:
      addresses:
        - 192.168.20.1/28
      routes:
        # Forward traffic from Laka side to Milu host, which bridges to DPU1 and onward to Hina
        - to: 192.168.10.0/28
          via: 192.168.20.1
    p0:
      optional: true
      addresses:
        - 192.168.20.2/28
      routes:
        # Route back toward Laka DPU and Laka host
        - to: 192.168.20.4/32
          via: 192.168.20.4
        - to: 192.168.20.7/32
          via: 192.168.20.4
    p1:
      optional: true
      addresses:
        - 192.168.20.3/28
      routes:
        - to: 192.168.20.5/32
          via: 192.168.20.3
        - to: 192.168.20.7/32
          via: 192.168.20.3
```

---

## Additional Resources

- [NVIDIA BlueField DPU Host-Side Interface Configuration](https://docs.nvidia.com/networking/display/bluefieldbsp4140/Host-side-Interface-Configuration)
- [NVIDIA DOCA SDK Documentation](https://docs.nvidia.com/doca/sdk/index.html)
- [NVIDIA Developer Portal](https://developer.nvidia.com/)
