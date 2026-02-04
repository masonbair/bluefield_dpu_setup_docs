# NVIDIA BlueField DPU Network Configuration Documentation

## Network Card Driver Installation

### If no ethernet: Ethernet Driver Setup

1. Download the appropriate Realtek ethernet driver from the [official Realtek downloads page](https://www.realtek.com/Download/List?cate_id=584)
2. Copy the driver to a USB drive
3. Install the driver on the target machine
     - This is done by running the autorun.sh script

### DOCA and MST Installation on Ubuntu Host

1. Follow the [official DOCA host installation guide](https://docs.nvidia.com/doca/sdk/doca-host+installation+and+upgrade/index.html)
2. Download the correct DOCA package for the host:
   - **Deployment Platform:** Host-Server
   - **Deployment Package:** DOCA-Host
   - **Target OS:** Linux (x86_64)
   - **Profile:** doca-all
   - **Distribution:** Ubuntu 22.04
   - **Installer Type:** deb_local
   - [Direct download link](https://developer.nvidia.com/doca-downloads?deployment_platform=Host-Server&deployment_package=DOCA-Host&target_os=Linux&Architecture=x86_64&Profile=doca-all&Distribution=Ubuntu&version=22.04&installer_type=deb_local)
3. **Important:** Follow all steps in the guide to load the driver and initialize MST

### BlueField DPU Firmware Installation

1. Follow the [BlueField Bundle Installation Guide](https://docs.nvidia.com/doca/sdk/bf-bundle-installation-and-upgrade/index.html)
2. Download the correct BlueField installation package:
   - **Deployment Platform:** BlueField
   - **Deployment Package:** BF-Bundle
   - **Distribution:** Ubuntu 24.04_64k
   - **Installer Type:** BFB
   - [Direct download link](https://developer.nvidia.com/doca-downloads?deployment_platform=BlueField&deployment_package=BF-Bundle&Distribution=Ubuntu&version=24.04_64k&installer_type=BFB)
3. Install the .bfb file for the BlueField
   - Choose "No pre-defined password" option (unless you want to set a password)
   - **Note:** The ISO file is not required

#### Troubleshooting rshim

If you encounter issues with rshim not being found:

```bash
sudo systemctl start rshim
sudo systemctl enable rshim
```

### MST Systemd Service Configuration

MST does not start automatically on reboot. To enable automatic startup:

1. Create a systemd service file for MST
2. Configure the service with:
   - `ExecStart`
   - `ExecStop`
   - `ExecRestart`
3. Find the MST binary path:
   ```bash
   which mst
   ```
   - Typically located at `/usr/bin/mst`
4. Build the file in `/etc/systemd/system/mst.service` on both hosts.
```
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

### Disabling Open vSwitch

Open vSwitch can create bridge interfaces that prevent proper communication between compute nodes. Disable it to avoid connectivity issues:

```bash
sudo systemctl stop openvswitch-switch
sudo systemctl disable openvswitch-switch
```

**Note:** This must be done on both of the DPUs to ensure proper network communication.

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

**Recommended:** For netplan automatic IP addresses and routing, reference to the netplan configuration files below

All Host-to-DPU communication uses the `192.168.100.x` subnet.

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

The DPU controller interface uses the `tmfifo_net0` network device.

1. Configure the tmfifo_net0 interface on the host:
   ```bash
   sudo ip addr add 192.168.100.1/30 dev tmfifo_net0
   ```

2. SSH into the DPU:
   ```bash
   ssh ubuntu@192.168.100.2
   ```
   - Default password: `ubuntu`
   - **Note:** Upon first login, you will be forced to set a new password (minimum 12 characters)
   - Current password: Set to the standard research machine password (entered twice during setup)

**Note:** You need to flush this ip address for the tmfifo_net0 device after you are done configuring it, as it interferes with the current IP addressing: `ip addr flush tmfifo_net0`

### Enabling IPv4 Routing on DPUs

Enable IP forwarding on both DPUs:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

To make this persistent across reboots, add to `/etc/sysctl.conf`:
```
net.ipv4.ip_forward=1
```
---

## Reference: Netplan Configuration Files

### Hina Host Configuration

**Note 1:** Once you have added each file in the correct folder, you will need to run the `netplan apply` command to activate the configuration.

**Note 2:** You may need to run a `sudo chmod 600 {name_of_configuration_file}` to remove the warning messages.

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
