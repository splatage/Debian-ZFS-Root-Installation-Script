# Debian Bookworm ZFS Root Installation Script

This repository provides a Bash script to automate the installation of Debian Bookworm with a ZFS root filesystem. It supports both BIOS (Legacy) and UEFI booting, and offers an option for an encrypted main ZFS pool (rpool).

---

## Table of Contents

* [Features](#features)
* [Prerequisites](#prerequisites)
* [Usage](#usage)
* [Important Considerations](#important-considerations)
* [How it Works](#how-it-works)
* [Contributing](#contributing)
* [License](#license)

---

## Features

* **Automated Disk Partitioning**: Automatically partitions selected disks with GPT.
* **ZFS Pool Creation**: Creates a separate ZFS pool for boot (`bpool`) and a main root pool (`rpool`).
* **Boot Type Support**: Configures the system for either BIOS (Legacy) or UEFI booting.
* **Optional Root Encryption**: Provides an option to encrypt the `rpool` for enhanced security.
* **Debian Bookworm Installation**: Uses `debootstrap` to install a minimal Debian Bookworm system.
* **Essential System Configuration**: Sets up hostname, network interfaces (by copying existing configuration), APT sources, locales, timezone, and keyboard layout.
* **ZFS Integration**: Installs ZFS packages, configures DKMS, and sets up ZFS services.
* **GRUB Installation**: Installs and configures GRUB for booting from ZFS, including necessary `initramfs` updates.
* **Basic Utilities**: Installs common utilities like `aptitude`, `vim`, `zsh`, `screen`, `tmux`, and `openssh-server`.
* **SSH Root Login**: Enables SSH login for the root user.
* **ZSH for Root**: Sets ZSH as the default shell for the root user.
* **Cleanup**: Unmounts filesystems and exports ZFS pools at the end of the installation.

---

## Prerequisites

* **Live Debian Environment**: This script is designed to be run from a Debian-based live environment (e.g., [Debian Live](https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/) (Standard) or a similar rescue disk).
* **Internet Connection**: An active internet connection is required for `debootstrap` and package installations.
* **`lsblk`**: Ensure `lsblk` is available for disk detection.
* **`sgdisk`**: Ensure `sgdisk` (from `gdisk` package) is available for partitioning.
* **`zfsutils-linux`**: ZFS utilities should be available or installable within your live environment if not already present, as the script will create ZFS pools.
* **Root Privileges**: The script must be run as root.

---

## Usage

1.  **Boot from a Debian Live or Rescue Environment**: Start your system from a Debian-based live USB or DVD.
2.  **Gain Root Access**: If you are not already root, switch to the root user:
    ```bash
    sudo -i
    ```
2.  **Install pre-requisites**: To install ZFS utilities:
    ```bash
    echo "deb http://deb.debian.org/debian bookworm main contrib non-free-firmware" >> /etc/apt/sources.list
    apt-get update
    apt install linux-headers-$(uname -r)
    apt install --yes debootstrap gdisk zfs-dkms zfsutils-linux grub2 
    ```
4.  **Download the Script**: Download the script to your live environment:
    ```bash
    wget https://raw.githubusercontent.com/danfossi/Debian-ZFS-Root-Installation-Script/refs/heads/main/debian_zfs_install.sh
    ```
5.  **Make it Executable**:
    ```bash
    chmod +x debian_zfs_install.sh
    ```
6.  **Run the Script**:
    ```bash
    ./debian_zfs_install.sh
    ```
7.  **Follow the Prompts**: The script will guide you through the following steps:
    * **Disk Selection**: Choose the physical disks where you want to install Debian ZFS. **Be extremely careful here, as all data on the selected disks will be ERASED.**
    * **Confirmation**: A strong warning will appear to confirm the destructive operation.
    * **RAID Level**: Choose the RAID level when multiple disks are selected.
    * **Booting Type**: Select between BIOS (Legacy) and UEFI booting.
    * **Encrypted Root Pool**: Decide if you want an encrypted `rpool`. If selected, you will be prompted for a passphrase later.
    * **Root Password**: You will be prompted to set the root password for the new system.
8.  **Reboot**: Once the script completes, remove the live installation medium and reboot your system.

---

## Important Considerations

* **Data Loss**: This script is **destructive**. It will **wipe all data** from the selected disks. **Proceed with extreme caution and ensure you have backups of any important data.**
* **Network Configuration**: The script copies the `/etc/network/interfaces` file from your live environment. Ensure your live system has a functional network configuration if you want it applied automatically to the installed system. Otherwise, you will need to configure networking manually after the first boot.
* **Localization**: The script sets the locale to `it_IT.UTF-8` and keyboard layout to `it`. You might need to adjust these settings within the chroot environment if you require a different localization.
* **GRUB Installation**: For UEFI, GRUB is installed with `--no-nvram` and `--bootloader-id=debian`. This means you might need to manually add an EFI boot entry in your motherboard's UEFI firmware if it's not automatically detected.
* **Single vs. Multiple Disks**: The script automatically configures ZFS pools as `mirror` if you select more than one disk.

---

## How it Works

The script operates in several distinct phases:

1.  **Disk Detection & Selection**: It first identifies available physical disks, excluding loop devices and CD-ROMs, and prompts the user to select the target disks.
2.  **Disk Preparation**: For each selected disk, it performs a complete wipe (`wipefs`, `blkdiscard`, `sgdisk --zap-all`, `dd`) to ensure a clean slate. It then creates a GPT partition table and specific partitions:
    * A BIOS boot partition (type `EF02`) for BIOS legacy booting, or an EFI System Partition (ESP, type `EF00`) for UEFI booting.
    * A 1GB ZFS partition (type `BF01`) for `bpool` (boot pool).
    * A ZFS partition (type `BF00`) for `rpool` (root pool), consuming the remaining space.
3.  **ZFS Pool Creation**:
    * It creates the `bpool` with `ashift=12`, `autotrim=on`, `compatibility=grub2`, and other ZFS properties optimized for a boot pool.
    * It creates the `rpool`, either encrypted with a passphrase or unencrypted, with `ashift=12`, `autotrim=on`, `acltype=posixacl`, `xattr=sa`, `dnodesize=auto`, `compression=lz4`, `normalization=formD`, and `relatime=on`.
    * If multiple disks are selected, `bpool` and `rpool` are created as ZFS mirrors.
4.  **Base System Installation**:
    * ZFS filesystems `rpool/ROOT/debian` and `bpool/BOOT/debian` are created.
    * `debootstrap` is used to install a minimal Debian Bookworm system into `/mnt`.
    * The `zpool.cache` file is copied to the new system.
5.  **System Configuration (Chroot)**: The script enters a `chroot` environment into `/mnt` to perform the following:
    * Update APT repositories.
    * Install essential packages (locales, console-setup) and configure them.
    * Install ZFS kernel modules (`zfs-initramfs`) and `linux-headers`/`linux-image`.
    * Install `grub-pc` (for BIOS) or `grub-efi-amd64` and `shim-signed` (for UEFI).
    * Set the root password.
    * Create and enable the `zfs-import-bpool.service` systemd unit for proper boot pool import.
    * Install additional utility packages.
    * Configure SSH for root login and set ZSH as the default root shell.
    * Update `GRUB_CMDLINE_LINUX` to include the ZFS root path.
    * Update `initramfs` and GRUB configuration.
    * Finally, install GRUB to the selected disks. For UEFI, it formats and mounts the EFI System Partition, then installs GRUB-EFI.
    * It adjusts ZFS cache file paths to remove the `/mnt` prefix after chroot operations.
6.  **Post-Chroot & Cleanup**: After exiting the chroot, the script unmounts all temporary mount points and exports the ZFS pools, leaving the system ready for reboot.

---

## Contributing

Contributions are welcome! If you find bugs, have suggestions for improvements, or want to add new features, please open an issue or submit a pull request.

---

## License

This project is licensed under the **GNU General Public License, Version 2**.

---
