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

On systems with **multiple DPUs**, each card is configured with its own unique tmfifo IP address by editing the netplan configuration inside the card via the rshim console. This avoids routing conflicts that would occur if all DPUs shared the same address.

#### Step 1: Open the rshim Console for the Target DPU

> **Machine:** Host machine (Laka, Hina, or Milu)

Connect to the DPU's serial console through its rshim device. For DPU 1 (rshim0):

```bash
sudo minicom -D /dev/rshim0/console
```

For DPU 2 (rshim1):

```bash
sudo minicom -D /dev/rshim1/console
```

Log in with the DPU's credentials when prompted.

#### Step 2: Configure a Unique tmfifo IP on the DPU

> **Machine:** Inside the DPU (via rshim console)

Each DPU must be assigned a unique tmfifo IP so that multiple DPUs can be accessed simultaneously without address conflicts. Edit the cloud-init netplan configuration file on the DPU:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Set the addresses according to which DPU you are configuring:

- **DPU 1 (rshim0):** use `192.168.101.1/30` on `tmfifo_net0`
- **DPU 2 (rshim1):** use `192.168.102.1/30` on `tmfifo_net1`

**Example configuration for DPU 1:**

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

After saving the file, apply the configuration:

```bash
sudo netplan apply
```

Repeat this process (Steps 1–2) for each DPU, adjusting the subnet accordingly.

#### Step 3: SSH into the DPU

> **Machine:** Host machine (Laka, Hina, or Milu) — this command connects you to the DPU

Once the tmfifo IP has been configured on the DPU, SSH into it using the corresponding address:

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
- Current password: Set to the standard research machine password (entered twice during setup)

---

### Multi-DPU SSH Workflow

> **Machine:** Milu host machine (or any host with more than one DPU)

Because each DPU has been configured with a unique tmfifo IP (via the rshim console in the steps above), multiple DPUs can be accessed simultaneously without routing conflicts or SSH host key collisions.

#### Workflow Summary for Milu (two DPUs)

**1. Configure a unique IP on each DPU via the rshim console** (see Steps 1–2 above). This only needs to be done once per card.

**2. SSH into each DPU directly using its dedicated address:**

For DPU 1:
```bash
ssh ubuntu@192.168.101.2
```

For DPU 2:
```bash
ssh ubuntu@192.168.102.2
```

No cleanup or host-key removal steps are required between connections since each DPU has a distinct address.


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
