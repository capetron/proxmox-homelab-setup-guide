# Proxmox Homelab Setup Guide

A complete Proxmox VE installation and homelab setup guide covering VM creation, LXC containers, networking, storage configuration, backup strategies, and clustering. From bare-metal install to production-ready virtualization platform.

## Table of Contents

- [Why Proxmox for Your Homelab](#why-proxmox-for-your-homelab)
- [Hardware Requirements](#hardware-requirements)
- [Installation](#installation)
- [Post-Install Configuration](#post-install-configuration)
- [Storage Configuration](#storage-configuration)
- [Networking Setup](#networking-setup)
- [Creating Virtual Machines](#creating-virtual-machines)
- [LXC Containers](#lxc-containers)
- [Backup and Disaster Recovery](#backup-and-disaster-recovery)
- [Clustering](#clustering)
- [GPU Passthrough](#gpu-passthrough)
- [Security Hardening](#security-hardening)
- [Monitoring and Maintenance](#monitoring-and-maintenance)
- [Common Homelab Services](#common-homelab-services)
- [Troubleshooting](#troubleshooting)
- [About Petronella Technology Group](#about-petronella-technology-group)

## Why Proxmox for Your Homelab

Proxmox Virtual Environment (VE) is a free, open-source Type 1 hypervisor built on Debian Linux with KVM virtualization and LXC containers. It provides an enterprise-grade web management interface without licensing costs, making it the ideal choice for homelabs, development environments, and small business infrastructure.

**Key advantages over alternatives:**

| Feature | Proxmox VE | VMware ESXi | Hyper-V |
|---------|-----------|-------------|---------|
| Cost | Free (optional subscription) | Free tier limited | Included with Windows Server |
| Web UI | Full-featured | vSphere Client required | WAC or SCVMM |
| Containers | Native LXC | No | No |
| ZFS | Built-in | No | No |
| Clustering | Free (up to 32 nodes) | vCenter license required | Failover Clustering |
| GPU Passthrough | Yes | Limited | Limited |
| Backup | Built-in (PBS integration) | Requires 3rd party | Requires 3rd party |

## Hardware Requirements

### Minimum (Single Node Homelab)

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 64-bit with VT-x/AMD-V | Intel i5/Ryzen 5 or better, 8+ cores |
| RAM | 8 GB | 32-64 GB ECC |
| Boot drive | 32 GB SSD | 256 GB NVMe |
| Storage | 256 GB | 1+ TB NVMe for VMs, HDD array for bulk |
| Network | 1 GbE | 2.5 GbE or 10 GbE |

### Recommended Hardware Platforms

- **Mini PCs:** Intel NUC, Minisforum MS-01, Beelink SER series
- **Workstations:** Dell OptiPlex Micro, HP EliteDesk, Lenovo ThinkCentre
- **Servers:** Dell PowerEdge T series, HP ProLiant MicroServer
- **Custom:** Any x86_64 system with IOMMU support

## Installation

### Step 1: Download and Create Boot Media

```bash
# Download Proxmox VE ISO from https://www.proxmox.com/en/downloads
# Create bootable USB with Ventoy, Rufus, or dd
dd if=proxmox-ve_8.x.iso of=/dev/sdX bs=4M status=progress
```

### Step 2: Install Proxmox VE

1. Boot from the USB drive
2. Select "Install Proxmox VE (Graphical)"
3. Accept the EULA
4. Select the target disk (choose ZFS RAID if using multiple disks)
5. Set country, timezone, and keyboard layout
6. Set the root password and admin email
7. Configure the management network interface, hostname (e.g., `pve.homelab.local`), IP address, gateway, and DNS

### Step 3: First Login

Access the web UI at `https://<ip-address>:8006`. Log in with `root` and the password you set during installation.

## Post-Install Configuration

### Remove Enterprise Repository (If No Subscription)

```bash
# Disable enterprise repository
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repository
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update
apt update && apt full-upgrade -y
```

### Remove Subscription Nag (Optional)

```bash
# This removes the "No valid subscription" popup
sed -Ezi.bak "s/(Ext\.Msg\.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

### Install Useful Packages

```bash
apt install -y vim htop iotop tmux lm-sensors smartmontools ethtool
sensors-detect --auto
```

### Configure Email Notifications

```bash
apt install -y libsasl2-modules mailutils
# Configure /etc/postfix/main.cf for SMTP relay (Gmail, Mailgun, etc.)
# This enables email alerts for backup completion, disk failures, etc.
```

## Storage Configuration

### ZFS Pool Setup

ZFS is the recommended filesystem for Proxmox -- it provides snapshots, compression, checksums, and RAID without a hardware controller.

```bash
# Create a mirrored ZFS pool (RAID 1 equivalent)
zpool create -f tank mirror /dev/sda /dev/sdb

# Create a RAIDZ1 pool (RAID 5 equivalent)
zpool create -f tank raidz1 /dev/sda /dev/sdb /dev/sdc

# Enable compression (recommended)
zfs set compression=lz4 tank

# Create datasets for VMs and containers
zfs create tank/vms
zfs create tank/containers
zfs create tank/backups
zfs create tank/isos
```

### Storage Types in Proxmox

| Storage Type | Use Case | Performance |
|-------------|----------|-------------|
| Local (ext4/xfs) | Boot drive, ISOs | Good |
| Local-ZFS | VM disks, containers | Excellent |
| NFS | Shared storage, backups | Good (network dependent) |
| Ceph | Distributed storage, HA | Excellent (3+ nodes) |
| LVM-Thin | VM disks, snapshots | Good |
| iSCSI | SAN storage | Excellent |

### Add Storage in Web UI

1. Datacenter > Storage > Add
2. Select storage type (Directory, ZFS, NFS, etc.)
3. Configure the path, content types (ISO, VZDump, Disk images), and nodes

## Networking Setup

### Bridge Configuration (Default)

Proxmox creates `vmbr0` during installation. This bridges VMs to the physical network.

```bash
# /etc/network/interfaces
auto lo
iface lo inet loopback

auto enp0s31f6
iface enp0s31f6 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.10.10.50/24
    gateway 10.10.10.1
    bridge-ports enp0s31f6
    bridge-stp off
    bridge-fd 0
```

### VLAN Configuration

```bash
# Create a VLAN-aware bridge
auto vmbr0
iface vmbr0 inet static
    address 10.10.10.50/24
    gateway 10.10.10.1
    bridge-ports enp0s31f6
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

Then assign VLANs per VM in the VM's network settings (e.g., VLAN tag 100 for a management VLAN).

### Dedicated Management Network

For production homelabs, dedicate a separate NIC for management traffic:

```bash
auto vmbr1
iface vmbr1 inet static
    address 10.10.20.50/24
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

## Creating Virtual Machines

### Upload an ISO

1. Navigate to the local storage > ISO Images > Upload
2. Or download directly: `wget -P /var/lib/vz/template/iso/ https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso`

### Create a VM via Web UI

1. Click "Create VM" in the top right
2. Configure:
   - **General:** VM ID, name
   - **OS:** Select ISO, set guest OS type
   - **System:** SCSI controller (VirtIO SCSI single), QEMU Agent enabled, EFI disk (for UEFI boot)
   - **Disks:** VirtIO Block, size as needed, enable Discard and SSD emulation for SSDs
   - **CPU:** Type = host (best performance), cores as needed
   - **Memory:** Ballooning enabled, set minimum and maximum
   - **Network:** VirtIO (paravirtualized), bridge = vmbr0

### Create a VM via CLI

```bash
qm create 100 \
  --name ubuntu-server \
  --memory 4096 \
  --cores 4 \
  --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --scsi0 local-zfs:32,discard=on \
  --cdrom local:iso/ubuntu-24.04-live-server-amd64.iso \
  --boot order=scsi0\;ide2 \
  --ostype l26 \
  --agent enabled=1
```

### Cloud-Init Templates

```bash
# Download cloud image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Create a VM and import the disk
qm create 9000 --name ubuntu-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9000 noble-server-cloudimg-amd64.img local-zfs
qm set 9000 --scsihw virtio-scsi-single --scsi0 local-zfs:vm-9000-disk-0
qm set 9000 --ide2 local-zfs:cloudinit
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1

# Configure cloud-init defaults
qm set 9000 --ciuser admin --cipassword <password> --sshkeys ~/.ssh/authorized_keys
qm set 9000 --ipconfig0 ip=dhcp

# Convert to template
qm template 9000

# Clone from template
qm clone 9000 101 --name my-server --full
```

## LXC Containers

LXC containers share the host kernel and use significantly less resources than full VMs. Use them for lightweight services.

### When to Use Containers vs. VMs

| Use Containers | Use VMs |
|---------------|---------|
| Web servers, databases | Windows workloads |
| DNS, DHCP, Pi-hole | Docker/Kubernetes |
| Monitoring (Grafana, Prometheus) | Firewalls (pfSense, OPNsense) |
| File servers (SMB, NFS) | Anything needing a different kernel |
| Home Assistant, Zigbee2MQTT | GPU passthrough workloads |

### Create a Container

```bash
# Download a template
pveam update
pveam available --section system
pveam download local debian-12-standard_12.2-1_amd64.tar.zst

# Create container
pct create 200 local:vztmpl/debian-12-standard_12.2-1_amd64.tar.zst \
  --hostname pihole \
  --memory 512 \
  --cores 1 \
  --net0 name=eth0,bridge=vmbr0,ip=10.10.10.53/24,gw=10.10.10.1 \
  --storage local-zfs \
  --rootfs local-zfs:8 \
  --unprivileged 1 \
  --features nesting=1
```

## Backup and Disaster Recovery

### Proxmox Backup Server (PBS)

PBS is the recommended backup solution for Proxmox. It provides deduplication, encryption, and incremental backups.

### Configure Scheduled Backups

1. Datacenter > Backup > Add
2. Schedule: Daily at 02:00
3. Selection: All VMs and containers (or specific)
4. Storage: Backup storage (PBS, NFS, or local)
5. Mode: Snapshot (no downtime)
6. Compression: ZSTD
7. Retention: Keep last 7 daily, 4 weekly, 3 monthly

### Backup via CLI

```bash
# Backup a single VM
vzdump 100 --storage backup-storage --mode snapshot --compress zstd

# Backup all VMs
vzdump --all --storage backup-storage --mode snapshot --compress zstd --mailto admin@example.com
```

### 3-2-1 Backup Strategy

- **3 copies** of data (production + 2 backups)
- **2 different media** (local SSD + NAS or PBS)
- **1 offsite copy** (cloud storage, remote PBS, or physically offsite)

## Clustering

### Create a Cluster (3+ Nodes Recommended)

```bash
# On the first node
pvecm create my-cluster

# On additional nodes
pvecm add <first-node-ip>

# Verify cluster status
pvecm status
```

### High Availability (HA)

```bash
# Enable HA for a VM
ha-manager add vm:100 --group ha-group --state started

# Check HA status
ha-manager status
```

## GPU Passthrough

### Enable IOMMU

```bash
# For Intel CPUs, edit /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# For AMD CPUs
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"

update-grub

# Add VFIO modules
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules

# Blacklist host GPU drivers
echo -e "blacklist nouveau\nblacklist nvidia\nblacklist radeon\nblacklist amdgpu" > /etc/modprobe.d/blacklist-gpu.conf

update-initramfs -u -k all
reboot
```

### Pass GPU to VM

```bash
# Find GPU PCI address
lspci -nn | grep -i vga

# Add PCI device to VM
qm set 100 --hostpci0 0000:01:00,pcie=1,x-vga=1
```

## Security Hardening

- [ ] **Change the default SSH port** -- Edit `/etc/ssh/sshd_config`
- [ ] **Disable root SSH login** -- Use a regular user with sudo
- [ ] **Enable firewall** -- Datacenter > Firewall > Enable. Create rules for management (port 8006) and SSH
- [ ] **Enable 2FA** -- Datacenter > Permissions > Two Factor > Add TOTP
- [ ] **Keep Proxmox updated** -- `apt update && apt full-upgrade` regularly
- [ ] **Use HTTPS with valid certificates** -- Configure Let's Encrypt or custom certificates
- [ ] **Restrict API access** -- Create API tokens with minimal permissions for automation
- [ ] **Monitor audit logs** -- Review `/var/log/pveproxy/access.log` and authentication logs

## Monitoring and Maintenance

### Built-in Monitoring

Proxmox provides CPU, memory, disk, and network graphs per node, VM, and container in the web UI.

### External Monitoring

```bash
# Install Prometheus node exporter on Proxmox host
apt install prometheus-node-exporter

# Proxmox VE Exporter for Prometheus
# https://github.com/prometheus-pve/prometheus-pve-exporter
```

### Maintenance Tasks

- [ ] Weekly: Check ZFS pool status (`zpool status`), review backup logs
- [ ] Monthly: Apply security updates, check disk SMART data (`smartctl -a /dev/sda`)
- [ ] Quarterly: Test backup restoration, review firewall rules, clean old ISOs and snapshots

## Common Homelab Services

| Service | Type | Resources |
|---------|------|-----------|
| Pi-hole / AdGuard Home | LXC | 512 MB RAM, 1 core |
| Home Assistant | VM or LXC | 2 GB RAM, 2 cores |
| Plex / Jellyfin | VM | 4 GB RAM, 4 cores + GPU |
| Nextcloud | LXC | 2 GB RAM, 2 cores |
| Grafana + Prometheus | LXC | 2 GB RAM, 2 cores |
| WireGuard / Tailscale | LXC | 256 MB RAM, 1 core |
| Nginx Proxy Manager | LXC | 512 MB RAM, 1 core |

## Troubleshooting

### VM Won't Start

```bash
# Check VM status and errors
qm status 100
journalctl -u pve-guests@100 --no-pager -n 50
```

### ZFS Pool Degraded

```bash
# Check pool status
zpool status
# Replace failed disk
zpool replace tank /dev/old-disk /dev/new-disk
```

### Web UI Not Accessible

```bash
# Check pveproxy service
systemctl status pveproxy
systemctl restart pveproxy
# Check if port 8006 is listening
ss -tlnp | grep 8006
```

## Additional Resources

- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [Proxmox Community Forum](https://forum.proxmox.com/)
- [Proxmox VE Wiki](https://pve.proxmox.com/wiki/)

## Contributing

Contributions are welcome. Please open an issue or submit a pull request with improvements, additional configurations, or corrections.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---

## About Petronella Technology Group

This guide is maintained by [Petronella Technology Group, Inc.](https://www.petronellatech.com/) -- a cybersecurity and IT services firm specializing in infrastructure management, virtualization, compliance (CMMC, HIPAA, SOC 2, NIST), and managed IT for businesses across the United States.

- Website: [https://www.petronellatech.com](https://www.petronellatech.com/)
- Book a consultation: [https://book.petronella.ai](https://book.petronella.ai/)
- Phone: [(919) 830-9435](tel:9198309435)
- LinkedIn: [Petronella Technology Group](https://www.linkedin.com/company/petronella-computer-consultants-inc-)
