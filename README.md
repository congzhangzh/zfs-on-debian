# Debian ZFS VPS Setup

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Debian](https://img.shields.io/badge/Debian-12%2C13-blue.svg)](https://www.debian.org/)
[![ZFS](https://img.shields.io/badge/ZFS-optimized-green.svg)](https://openzfs.org/)

**Simplified and optimized scripts to install Debian with ZFS root filesystem on VPS platforms.**

Focused on **Debian systems** with intelligent VPS provider detection and simplified ZFS architecture for better performance and easier maintenance.

> ⚠️ **WARNING:** All data on the target disk will be completely destroyed during installation.

## ✨ Key Features

- 🎯 **Debian-focused**: Optimized specifically for Debian 10, 11, and 12
- 🔧 **Simplified ZFS structure**: Reduced complexity while maintaining ZFS benefits  
- 🌐 **Multi-VPS support**: Auto-detection for Hetzner, Netcup, and generic providers
- ⚡ **Performance optimized**: LZ4 compression, intelligent swap sizing
- 📦 **Minimal complexity**: 75% fewer ZFS filesystems compared to complex setups
- 🚀 **Fast deployment**: Streamlined installation process

## 🚀 Quick Start

### Primary Support: Debian 12 (Recommended)

```bash
wget -qO- https://raw.githubusercontent.com/congzhangzh/zfs-on-debian/main/debian-zfs-setup.sh | bash -

# Or clone and run locally (recommended for customization):
git clone https://github.com/congzhangzh/zfs-on-debian.git
cd zfs-hetzner-vm
./debian-zfs-setup.sh
```

### Legacy Debian Versions

<details>
<summary>Click to expand legacy options</summary>

**Debian 11:**
```bash
wget -qO- https://raw.githubusercontent.com/YOUR_USERNAME/zfs-hetzner-vm/main/hetzner-debian11-zfs-setup.sh | bash -
```

**Debian 10:**
```bash
wget -qO- https://raw.githubusercontent.com/YOUR_USERNAME/zfs-hetzner-vm/main/hetzner-debian10-zfs-setup.sh | bash -
```

</details>

## 📋 Prerequisites

### VPS Requirements
- **Memory**: Minimum 1GB, recommended 2GB+
- **Storage**: 20GB+ available space
- **Network**: Stable internet connection
- **Access**: Root access via rescue system

### Supported VPS Providers
- ✅ **Hetzner** (Cloud & Dedicated) - Full optimization
- ✅ **Netcup** (VPS & Root Server) - Native support  
- ✅ **Generic KVM providers** - Standard compatibility

## 🛠️ Installation Steps

### 1. Boot into Rescue System
For **Hetzner**:
TODO: check the router config part?
- Login to cloud console
- Navigate to "Rescue" tab
- Add your SSH public key
- Select "linux64" as rescue OS
- Click "Enable rescue and power cycle"

For **Netcup**:
- Access VPS control panel
- Enable rescue system
- Configure SSH key access
- Reboot into rescue mode

### 2. Run Installation (Recommended Method)
```bash
# Start screen session for network reliability
screen -S zfs-install

# Method 1: Direct download and run (replace YOUR_USERNAME)
wget -qO- https://raw.githubusercontent.com/congzhangzh/zfs-on-debian/main/debian-zfs-setup.sh | bash -

# Method 2: Clone and run locally (better for customization)
git clone https://github.com/congzhangzh/zfs-on-debian.git
cd zfs-hetzner-vm
./debian-zfs-setup.sh
```

### 3. Follow Interactive Prompts
- **Hostname**: Enter desired system hostname
- **Swap size**: Default is 2x memory (automatically calculated)
- **Disk selection**: Choose target disks for installation
- **Encryption**: Optional ZFS encryption setup

### 4. Wait for Completion
- Installation typically takes 15-30 minutes
- System will automatically reboot when finished
- Login with the same SSH key used in rescue system

## 🏗️ Architecture Overview

### Simplified ZFS Structure
```
📁 Boot Pool (bpool)
└── bpool/BOOT/debian          # Kernel, initrd, GRUB files

📁 Root Pool (rpool)  
├── rpool/ROOT/debian          # Main system (/, /home, /var, etc.)
└── rpool/swap                 # Swap partition (optional)
```

### Key Optimizations
- **75% fewer filesystems** compared to complex ZFS setups
- **LZ4 compression** for optimal performance/space balance
- **Legacy mounting** for reliable systemd integration
- **Provider-specific networking** for better compatibility

## 🔧 Advanced Configuration

### Custom Installation Options
```bash
# Clone repository for customization
git clone https://github.com/congzhangzh/zfs-on-debian.git
cd zfs-on-debian

# Edit configuration variables at the top of the script
nano debian-zfs-setup.sh

# Review changes and run customized installation
./debian-zfs-setup.sh
```

### Post-Installation Management
```bash
# Check ZFS status
zpool status
zfs list

# Create snapshots
zfs snapshot rpool/ROOT/debian@backup-$(date +%Y%m%d)

# Monitor ZFS performance
zpool iostat 1
```

## 🆘 Troubleshooting

### Common Issues

**Network timeout during installation:**
```bash
# Use screen session and check network
screen -S zfs-install
ping -c 4 8.8.8.8
```

**Boot failure after installation:**
```bash
# Boot into rescue system and check ZFS pools
zpool import -f rpool
zfs mount rpool/ROOT/debian
```

**Space usage concerns:**
```bash
# Check compression efficiency
zfs get compressratio rpool
zfs list -o space
```

### Getting Help
- 📖 Check the [technical report](tech_report.md) for detailed analysis
- 🐛 Open an issue for bugs or feature requests
- 💬 Discussions for general questions and community support

## 📚 Documentation

- [Technical Report](tech_report.md) - In-depth analysis and optimization details
- [Acknowledgments](thanks.md) - Credits and project history
- [Development Notes](dev_flow_in_china.md) - Development workflow information

## 🤝 Contributing

This project focuses on Debian systems and VPS optimization. Contributions are welcome for:
- Bug fixes and improvements
- Additional VPS provider support
- Documentation enhancements
- Performance optimizations

## 📄 License

MIT License - see the original [terem42/zfs-hetzner-vm](https://github.com/terem42/zfs-hetzner-vm) project for details.

## 🙏 Acknowledgments

This project is a Debian-focused fork of the excellent [terem42/zfs-hetzner-vm](https://github.com/terem42/zfs-hetzner-vm). 

All credit for the original concept and implementation goes to the original authors and contributors.
