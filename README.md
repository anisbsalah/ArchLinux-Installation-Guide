# Arch Linux Installation Guide

## Introduction

### Why this guide?

The Arch Wiki has this information spread across multiple pages, and I think this is much more clearly laid out and straight forward.
It also contains some packages and decisions that are personal preference.

### Important other resources

The Arch Wiki is a very powerful resource. If you have any problems it's the first place to search for solutions: <https://wiki.archlinux.org>

Especially for the installation please read: <https://wiki.archlinux.org/index.php/installation_guide>

Sometimes packages need manual intervention which is announced at <https://www.archlinux.org/news/>

This tutorial is inspired by <https://arch.d3sox.me/> and have taken out most stuff of it.

## 1- Pre-installation

### 1.1 Prepare the installation media

[Download](https://www.archlinux.org/download/) the latest Arch Linux ISO version and create a bootable USB drive.
Make sure you enable USB boot in BIOS settings and set USB Disk as boot priority.

### 1.2 BIOS Configuration for UEFI

Before booting from an usb stick you better check your hardware settings. Boot into your hardware settings or BIOS or UEFI settings. Then check following settings. It can have different names and different keyboard shortcuts to reach it.

- **Set Boot Mode to UEFI**. Most modern systems use UEFI, so it's generally on by default.

- **Disable Secure Boot**. If secure boot is enabled it must be turned off since Linux boot loaders don't typically have digital signatures. Note that if you intend on running a dual-boot system with Windows and Linux you won't be able to use disk encryption on the partition containing Windows, as it requires secure boot.

- **Disable Launch CSM or Legacy Support**.

- **Disable Fast Startup Mode**. If you are dual booting with Windows turn off Fast Startup. This feature puts Windows into hibernation when you power off. Because some systems are still active during hibernation, booting into Linux can cause various nasty problems.

### 1.3 Boot the live environment

After booting Arch from the USB drive...

### 1.4 Set the console keyboard layout

The default keymap is US. Available layouts can be listed with:

```bash
localectl list-keymaps
```

or

```bash
find /usr/share/kbd/keymaps/ -type f | more
```

⌨️ Set your keymap (e.g. `fr`) with:

```bash
loadkeys fr
```

### 1.5 Set the console font

Available fonts can be listed with:

```bash
ls /usr/share/kbd/consolefonts/
```

🔤 Set the console font with:

```bash
setfont ter-v18b
```

### 1.6 Verify the boot mode

```bash
ls /sys/firmware/efi/efivars
```

or

```bash
efivar -l
```

If the directory does not exist, the system may be booted in Legacy BIOS Mode.
Most likely you want to do a UEFI install so please double-check if your system supports UEFI and you selected the correct entry in the boot menu (In most cases prefixed with UEFI).

### 1.7 Connect to the internet

- Ensure your network interface is listed and enabled, for example with:

```bash
ip link
```

- For wireless and WWAN, make sure the card is not blocked with `rfkill`.

```bash
rfkill unblock wlan
```

- Connect to the network:

  - Ethernet: plug in the cable.

  - Wi-Fi: authenticate to the wireless network using `iwctl`.

```bash
iwctl
device list
# Your device name might be different (replace wlan0)
station wlan0 scan
station wlan0 get-networks
# Replace <SSID> with your network name from the previous command
station wlan0 connect <SSID>
exit
```

#### Check the internet connection with `ping`

```bash
ping -c 4 archlinux.org
```

### 1.8 Update the system clock

🕒 Ensure the system clock is accurate:

```bash
timedatectl set-ntp true
```

🌍 Set your time zone:

```bash
timedatectl set-timezone Africa/Tunis
```

### 1.9 Partition the disks

> _Tip:
> When recognized by the live system, disks are assigned to a block device such as /dev/sda, /dev/nvme0n1 or /dev/mmcblk0._

#### 1.9.1 Identify the devices

To get an overview use `fdisk` or `lsblk` to list your partition table and find out the device you want to use.

```bash
fdisk -l
```

or

```bash
lsblk
```

#### 1.9.2 Delete all the existing partitions on the disk to be used

```bash
wipefs -a /dev/the_disk_to_be_partitioned
```

If that is not working for you, you can try this one as well:

```bash
sgdisk -Z /dev/the_disk_to_be_partitioned
```

#### 1.9.3 Start partitioning tool

▶️ Text-based

```bash
fdisk /dev/the_disk_to_be_partitioned
```

▶️ UEFI only text-based

```bash
gdisk /dev/the_disk_to_be_partitioned
```

▶️ Graphical (Recommended for beginners)

```bash
cfdisk /dev/the_disk_to_be_partitioned
```

▶️ UEFI only Graphical (Recommended for beginners)

```bash
cgdisk /dev/the_disk_to_be_partitioned
```

#### 1.9.4 Decide partition table type and create partitions

- BIOS: You can use both GPT and MBR (DOS)
- UEFI: You need to use GPT

##### DOS/MBR (BIOS)

| Needed | Partition | Partition type   | Code (fdisk) | Flags        | Mount point |
| ------ | --------- | ---------------- | ------------ | ------------ | ----------- |
| ❌     | /dev/sdXY | Linux swap       | 82           | -            | -           |
| ✔️     | /dev/sdXY | Linux filesystem | 83           | Bootable (a) | /mnt        |
| ❌     | /dev/sdXY | Linux filesystem | 83           | -            | /mnt/home   |

##### GPT (BIOS)

| Needed | Partition | Partition type   | Code (fdisk/gdisk) | Mount point |
| ------ | --------- | ---------------- | ------------------ | ----------- |
| ✔️     | /dev/sdXY | BIOS boot        | 4 / ef02           | -           |
| ❌     | /dev/sdXY | Linux swap       | 82 / 8200          | -           |
| ✔️     | /dev/sdXY | Linux filesystem | 83 / 8300          | /mnt        |
| ❌     | /dev/sdXY | Linux filesystem | 83 / 8300          | /mnt/home   |

##### GPT (UEFI)

| Needed | Partition | Partition type   | Code (fdisk/gdisk) | Mount point   |
| ------ | --------- | ---------------- | ------------------ | ------------- |
| ✔️     | /dev/sdXY | EFI partition    | 1 / ef00           | /mnt/boot/efi |
| ❌     | /dev/sdXY | Linux swap       | 82 / 8200          | -             |
| ✔️     | /dev/sdXY | Linux filesystem | 83 / 8300          | /mnt          |
| ❌     | /dev/sdXY | Linux filesystem | 83 / 8300          | /mnt/home     |

###### Size recommandations

**\* EFI partition:**

- At least 300 MiB.
- If multiple kernels will be installed, then no less than 1 GiB.

**\* BIOS boot:**

- 1MB (unformatted)

**\* Swap:**

| System RAM | Recommended swap space | Swap space if using hibernation |
| ---------- | ---------------------- | ------------------------------- |
| < 2GB      | 2x the amount of RAM   | 3x the amount of RAM            |
| 2-8GB      | Equal to amount of RAM | 2x the amount of RAM            |
| 8-64GB     | At least 4GB           | 1.5x the amount of RAM          |
| 64GB       | At least 4GB           | Hibernation not recommended     |

### 1.10 Format partitions

#### Root partition filesystem

💽 This will create the file system where the system will be installed on.

To create a btrfs file system on `/dev/root_partition`, run

```bash
mkfs.btrfs -f /dev/root_partition
```

#### EFI system partition

If you created an EFI system partition, format it to FAT32 using `mkfs.fat`

```bash
mkfs.fat -F32 /dev/efi_system_partition
```

#### Swap partition

If you created a partition for swap, initialize it with `mkswap`:

```bash
mkswap /dev/swap_partition
```

#### Create subvolumes to better organize our data and to exclude them from btrfs snapshots

```bash
mount /dev/root_partition /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@srv
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap
```

### 1.11 Mount the file systems

#### Mount the root volume to /mnt

Let us unmount /mnt and remount all the subvolumes with options. For example if the root volume is `/dev/root_partition` :

```bash
umount /mnt
mount -o defaults,noatime,compress=zstd,subvol=@ /dev/root_partition /mnt
mkdir -p /mnt/{home,root,srv,var/log,var/cache,tmp,.snapshots,swap}
mount -o defaults,noatime,compress=zstd,subvol=@home /dev/root_partition /mnt/home
mount -o defaults,noatime,compress=zstd,subvol=@root /dev/root_partition /mnt/root
mount -o defaults,noatime,compress=zstd,subvol=@srv /dev/root_partition /mnt/srv
mount -o defaults,noatime,compress=zstd,subvol=@log /dev/root_partition /mnt/var/log
mount -o defaults,noatime,compress=zstd,subvol=@cache /dev/root_partition /mnt/var/cache
mount -o defaults,noatime,compress=zstd,subvol=@tmp /dev/root_partition /mnt/tmp
mount -o defaults,noatime,compress=zstd,subvol=@snapshots /dev/root_partition /mnt/.snapshots
mount -o defaults,noatime,subvol=@swap /dev/root_partition /mnt/swap
```

#### Create any remaining mount points (such as /mnt/boot) and mount the volumes in their corresponding hierarchical order

```bash
mkdir /mnt/boot
mount /dev/boot_partition /mnt/boot
```

or shortly:

```bash
mount --mkdir /dev/boot_partition /mnt/boot
```

▶️ For UEFI systems, mount the EFI system partition:

```bash
mkdir -p /mnt/boot/efi
mount /dev/efi_system_partition /mnt/boot/efi
```

or shortly:

```bash
mount --mkdir /dev/efi_system_partition /mnt/boot/efi
```

#### Swap

- If you created a swap volume, enable it with swapon:

```bash
swapon /dev/swap_partition
```

- As an alternative to creating an entire partition, a swap file offers the ability to vary its size on-the-fly, and is more easily removed altogether. To create a swap file:

```bash
### Ram-size in bytes:
RAM_SIZE=$(free -b -t | awk 'NR == 2 {print $2}')
### Create a swap file equal to the ram size:
btrfs filesystem mkswapfile --size $RAM_SIZE --uuid clear /mnt/swap/swapfile
### When activating the swap file you can use -p or --priority flag to set the swap file priority, by default it is -2
swapon -p 100 /mnt/swap/swapfile
```

## 2- Installation

### 2.1 Select the mirrors

You can select the mirrors that are closest to you geographically. BUT it does not mean they will be the fastest. Normally I will change nothing until the speed is not satisfactory. On the live system, after connecting to the internet, reflector updates the mirror list by choosing 20 most recently synchronized HTTPS mirrors and sorting them by download rate. Read the info about mirror servers as this is quite important.

```bash
reflector --latest 20 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
```

### 2.2 Install essential packages

⏳ This will install the system and may take a while.

- Enable parallel downloads for faster installation:

```bash
sed -i 's/^#ParallelDownloads/ParallelDownloads/' /etc/pacman.conf
```

> ⚠️ Warning: To ensure system stability append the microcode package for your CPU to the following command.
>
> - amd-ucode for AMD CPUs
> - intel-ucode for Intel CPUs

See <https://wiki.archlinux.org/index.php/Microcode>

```bash
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers intel-ucode btrfs-progs sudo nano bash-comlpetion terminus-font
```

## 3- Configure the system

### 3.1 Create the filesystem table (Fstab)

This will create the filesystem table which contains all the partitions and mountpoints:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Edit the fstab configuration to add an entry for the swap file (if created):

```bash
echo '/swap/swapfile none swap defaults 0 0' >>/mnt/etc/fstab
```

### 3.2 Chroot

After you entered this command, you are basically in the installed system.

```bash
arch-chroot /mnt
```

### 3.3 Time

📅 Set the time zone (You can tab-complete your stuff after typing zoneinfo):

```bash
ln -sf /usr/share/zoneinfo/Africa/Tunis /etc/localtime
hwclock --systohc
```

### 3.4 Localization

🌐 Edit `/etc/locale.gen` and uncomment (remove the # in front of) all languages you need:

```bash
nano /etc/locale.gen
```

🏁 Generate locales

```bash
locale-gen
```

🔘 Set the system language

```bash
echo 'LANG=en_US.UTF-8' >> /etc/locale.conf
```

```bash
echo 'LC_ADDRESS=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_IDENTIFICATION=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_MEASUREMENT=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_MONETARY=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_NAME=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_NUMERIC=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_PAPER=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_TELEPHONE=en_US.UTF-8' >> /etc/loacle.conf
echo 'LC_TIME=en_US.UTF-8' >> /etc/loacle.conf
```

⌨️ Set tty keymap ( e.g. `fr`) and console font ( e.g. `ter-v18b`)

```bash
echo 'KEYMAP=fr' >> /etc/vconsole.conf
echo 'FONT=ter-v18b' >> /etc/vconsole.conf
```

### 3.5 Network configuration

📛 Create the hostname file. The hostname will be the name of your PC on your network.

```bash
echo 'ArchLinux' >> /etc/hostname
# Edit /etc/hosts file using echo command or open it with nano and add the lines:
echo '127.0.0.1		localhost' >> /etc/hosts
echo '::1	        localhost' >> /etc/hosts
echo '127.0.1.1		ArchLinux.localdomain	ArchLinux' >> /etc/hosts
```

📶 Complete the network configuration for the newly installed environment. That may include installing suitable network management software.

We will use the application `NetworkManager` afterwards in any of our desktops we would like to install. Therefore it makes sense to install and activate this application right now. Then, you can use `nmtui` or `wifi-menu` to configure your network profile.

Make sure you type `NetworkManager` in the second line. It is case sensitve. You will get 3 symlink lines as a result. When we boot we will have internet on board.

```bash
pacman -S networkmanager
systemctl enable NetworkManager
```

🔴 Install necessary drivers for wireless card if needed (e.g. for my Broadcom BCM4312 card)

```bash
pacman -S --needed dkms
pacman -S --needed broadcom-wl-dkms
#pacman -S --needed broadcom-wl
```

To find out what hardware you have in your computer:

```bash
lspci -v | grep -i network
```

### 3.6 Initramfs

Modify `mkinitcpio.conf` :

```bash
nano /etc/mkinitcpio.conf
```

- Add `btrfs` to MODULES
- Add `setfont` to BINARIES
- Replace `keymap` and `consolefont` HOOKS with `sd-vconsole`
- Uncomment the following line: `#COMPRESSION="ZSTD"`

And recreate the initramfs image with:

```bash
mkinitcpio -P
```

### 3.7 Root password

🔑 Use a strong and complicated password

```bash
passwd
```

### 3.8 Bootloader

#### UEFI

```bash
pacman -S --needed grub os-prober efibootmgr mtools dosfstools
# Change the directory to /boot if you mounted the EFI partition at /boot
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

#### BIOS

```bash
pacman -S --needed grub os-prober mtools dosfstools
# Replace sdX with your disk name, not the partition
grub-install --target=i386-pc /dev/sdX
grub-mkconfig -o /boot/grub/grub.cfg
```

## 4 Reboot

```bash
exit
umount -R /mnt
reboot
```

**_And log in back with root_**.

## 5 Post-installation

### 5.1 System administration

#### 5.1.1 Users and groups

##### Add your user

👦 This will be the user you'll use to log in. For group reference see <https://wiki.archlinux.org/index.php/Users_and_groups#Group_list>

```bash
useradd -m -g users -G audio,video,network,wheel,storage,rfkill -s /bin/bash anisbsalah
passwd anisbsalah
```

##### Enable sudo

🧐 This will give your user administrative privileges.

```bash
EDITOR=nano visudo
```

⭐ Uncomment (remove the # in front of) the following line:

`%wheel ALL=(ALL:ALL) ALL`

**_And log in back with user account_**.

### 5.2 Package management

#### 5.2.1 Pacman

Edit pacman configuration file:

```bash
nano /etc/pacman.conf
```

##### a- Enable parallel downloads

🌐 Depending on your internet connection enabling parallel downloads may speed up the package download process.

⭐ Uncomment (remove the # in front of) this line and set it to your desired value:

```ini
#ParallelDownloads = 5
```

##### b- Extra candy

🍬 If you want some extra candy you can uncomment `Color` and `VerbosePkgLists` and add `ILoveCandy` under `Misc options`.

#### 5.2.2 Repositories

👾 `multilib` is a repository which contains 32-bit libraries and is disabled by default (needed for some games & software; highly recommended to enable).

If you plan on using 32-bit applications, you will want to enable the multilib repository.

⭐ Uncomment (remove the # in front of) the following lines:

```ini
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

##### Update the repos

```bash
sudo pacman -Syu
```

### 5.3 Graphical user interface

#### 5.3.1 Display server

Xorg is the public, open-source implementation of the X Window System (commonly X11, or X). It is required for running applications with graphical user interfaces (GUIs), and the majority of users will want to install it.

```bash
sudo pacman -S --needed xorg xorg-apps xorg-server xorg-xinit xorg-xkill xorg-xwayland
```

#### 5.3.2 Display drivers

The default modesetting display driver will work with most video cards, but performance may be improved and additional features harnessed by installing the appropriate driver for AMD or NVIDIA products.

> ⚠️ Warning: The linux kernel can handle many graphical cards, so try first without any extra installation.

To find out what hardware you have in your computer:

```bash
lspci -v | grep -e VGA -e 3D
```

##### Mesa (useful for all GPUs)

```bash
sudo pacman -S --nedded mesa lib32-mesa
```

##### Vulkan (useful for all GPUs)

```bash
sudo pacman -S --nedded vulkan-icd-loader lib32-vulkan-icd-loader
```

##### Open Source drivers

Only install this if you use an AMD or Intel GPU or want to use the open source NVIDIA driver (Nouveau, not developed by NVIDIA)

```bash
sudo pacman -S <driver>
```

- `xf86-video-amdgpu` is for newer AMD GPUs
- `xf86-video-nouveau` is the open source NVIDIA driver
- `xf86-video-intel` is the open source Intel driver (You probably want to leave this out, and it will use the modesetting driver. For more information refer to [the wiki](https://wiki.archlinux.org/index.php/Intel_graphics#Installation))
- `xf86-video-ati` is for older AMD GPUs
- `xf86-video-vmware` for VirtualBox, VMWare, QEMU
- `xf86-video-fbdev` for Hyper-V
- If you don't know it you can install all, but it could happen that the internal graphics card is used if you install the driver for it

##### AMD Utils

Only install these packages if you are using an AMD GPU

```bash
sudo pacman -S libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau libva-vdpau-driver lib32-libva-vdpau-driver vulkan-radeon lib32-vulkan-radeon
```

##### Intel Utils

Only install this package if you are using an Intel GPU

```bash
sudo pacman -S vulkan-intel
```

##### Nvidia proprietary driver

Only install these packages if you are using an NVIDIA GPU

```bash
sudo pacman -S nvidia nvidia-utils lib32-nvidia-utils libvdpau lib32-libvdpau
```

> ⚠️ Warning
> NVIDIA's Linux drivers have a bad reputation when it comes to stability and compatibility with all systems.
> If you experience any problems later on consult https://wiki.archlinux.org/title/NVIDIA for troubleshooting.

##### Early KMS start

Some systems require early KMS start to work properly. Read the [Arch Wiki entry](https://wiki.archlinux.org/title/Kernel_mode_setting#Early_KMS_start) about it

```bash
nano /etc/mkinitcpio.conf
```

Change `MODULES=()` to

- `MODULES=(amdgpu)` if you installed `xf86-video-amdgpu`
- `MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)` if you installed `nvidia`
- `MODULES=(i915)` if you are using Intel graphics
- for any other driver you can skip this step

Remove `kms` inside `HOOKS=()` if you installed `nvidia`

and run

```bash
mkinitcpio -P
```

#### 5.3.3 Desktop environment

```bash
sudo pacman -S plasma-desktop konsole kate dolphin firefox ssdm
```

```bash
sudo systemctl enable ssdm.service
```

#### 5.3.4 User directories

Well-known user directories like Downloads or Music are created by the xdg-user-dirs-update.service user service, that is provided by xdg-user-dirs and enabled by default upon install.

```bash
sudo pacman -S --needed xdg-user-dirs xdg-utils
```

> ⚠️ Note: The installation can just end here. Then you add more applications to have a system suited to your needs and theme it anyway you like.

### 5.4 Useful packages

#### AUR Setup

The Arch User Repository is a community-driven repository for Arch users. [yay](https://github.com/Jguer/yay) is a pacman wrapper that allows installing AUR packages.

```bash
pacman -S --noconfirm --needed git base-devel
git clone https://aur.archlinux.org/yay.git /tmp/yay
cd /tmp/yay
makepkg -si
```

[paru](https://github.com/Morganamilo/paru) is a good alternative to `yay`. It's easy to use due to its similarity with yay's CLI. To install it, simply execute the following commands:

```bash
pacman -S --noconfirm --needed git base-devel
git clone https://aur.archlinux.org/paru.git /tmp/paru
cd /tmp/paru
makepkg -si
```

#### Networking

```bash
sudo pacman -S --needed avahi curl bind bridge-utils dhclient dialog dnsutils dnsmasq gvfs gvfs-smb ifplugd inetutils ipset iptables-nft iw iwd mobile-broadband-provider-info modemmanager net-tools netctl networkmanager network-manager-applet networkmanager-openconnect networkmanager-openvpn networkmanager-pptp networkmanager-qt networkmanager-vpnc nfs-utils nm-connection-editor nss-mdns ntp openbsd-netcat openconnect openssh openvpn samba wireless-regdb wireless_tools wget wpa_supplicant
```

```bash
sudo systemctl enable avahi-daemon.service
sudo systemctl enable ntpd.service
sudo systemctl enable sshd.service
sudo systemctl enable wpa_supplicant.service
```

#### Sound

##### a- Pipewire

```bash
sudo pacman -S --needed alsa-card-profiles alsa-firmware alsa-lib alsa-plugins alsa-utils sof-firmware pipewire pipewire-alsa pipewire-audio pipewire-docs pipewire-ffado pipewire-jack pipewire-pulse pipewire-roc pipewire-v4l2 pipewire-x11-bell pipewire-zeroconf lib32-pipewire lib32-pipewire-jack wireplumber gst-plugin-pipewire gstreamer gst-plugins-good gst-plugins-bad gst-plugins-base gst-plugins-ugly pavucontrol playerctl volumeicon
```

See <https://wiki.archlinux.org/title/PipeWire>

##### b- PulseAudio

```bash
sudo pacman -S --needed alsa-card-profiles alsa-firmware alsa-lib alsa-plugins alsa-utils sof-firmware pulseaudio pulseaudio-alsa pulseaudio-bluetooth pulseaudio-equalizer pulseaudio-jack pulseaudio-lirc pulseaudio-rtp pulseaudio-zeroconf lib32-alsa-plugins lib32-libpulse gstreamer gst-plugins-good gst-plugins-bad gst-plugins-base gst-plugins-ugly pavucontrol playerctl volumeicon
```

See <https://wiki.archlinux.org/title/PulseAudio>

#### Bluetooth support

```bash
sudo pacman -S --needed bluez bluez-utils bluez-libs bluedevil
```

```bash
sudo systemctl enable bluetooth.service
```

#### Printer support

```bash
sudo pacman -S --needed cups cups-pdf cups-filters cups-pk-helper foomatic-db foomatic-db-engine foomatic-db-gutenprint-ppds foomatic-db-ppds foomatic-db-nonfree foomatic-db-nonfree-ppds ghostscript gsfonts gutenprint gtk3-print-backends libcups system-config-printer sane skanlite
```

```bash
sudo systemctl enable cups.service
```

#### Input drivers

```bash
sudo pacman -S --needed xf86-input-synaptics xf86-input-libinput xf86-input-evdev xf86-input-elographics
```

#### Power management

```bash
sudo pacman -S --needed acpi acpid acpi_call tlp
```

```bash
sudo systemctl enable acpid.service
sudo systemctl enable tlp.service
```

#### General packages, archive and file system utils

```bash
sudo pacman -S --needed arj bash-completion cabextract cifs-utils e2fsprogs exfat-utils filelight file-roller man-db man-pages nfs-utils nilfs-utils ntfs-3g p7zip rar reflector rsync sharutils squashfs-tools terminus-font texinfo udiskie udisks2 unace unarchiver unrar unzip usbutils uudeview xz zip
```

```bash
sudo systemctl enable reflector.timer
```
