# Final Guide

## Initial Install

### Windows setup
* Disable Secure Boot
* Go to `regedit.msc` and go to `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation`. Add a new QWORD called `RealTimeIsUniversal` and give it a value of `1`. You do not have to touch the Settings App for time and date
* [Disable Fast Startup](https://www.lifewire.com/disable-fast-startup-in-windows-10-5094422)
* [Disable Hibernation](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/disable-and-re-enable-hibernation) (`powercfg.exe /hibernate` as admin in Command Prompt)
* Decrease size of C: Drive in `Create and Format Hard Disk Partitions` to the amount of empty space you want for your arch installation

#### Other Windows Things
* Refer to [this ArchWiki guide for bluetooth after installation](https://wiki.archlinux.org/title/Bluetooth#Dual_boot_pairing)
* If you would like Windows Hard Disk Encryption refer to [this guide](https://octetz.com/docs/2020/2020-2-16-arch-windows-install/) which will help setup Veracrypt and Grub
* Intel Rapid Storage Technology (IRST) can be disabled from Intel Optane Memory and Storage Management App (if it cannot be disabled in bios). if IRST was included when the machine was purchased, the app should be preinstalled as well. You should have the option to disable it.

### Connect to internet, network time synchronization, setup ssh
```bash
iwctl
[iwd]# device list
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station device connect Network_Name
ping archlinux.org

timedatectl set-ntp true

passwd
systemctl start sshd
ip a # Use local ip (192.168.[num].[num])
# From other computer: ssh root@ip
```

### Partition disk, encrypt root, format boot and cryptroot
```bash
lsblk # fdisk -l
cgdisk /dev/nvme0n1
# Go to free space sector and select [ New ]
#   * Default first sector
#   * 512MiB Size
#   * Default partition type (linux filesystem)
#   * Name of "boot"
# Remaining space, select [ New ]
#   * Default options for 1 through 3
#   * Name of "root"
# Click on [ Write ] and type yes to confirm
lsblk # fdisk -l Note the EFI (likely 1), boot (likely 5), and root (likely 6)

cryptsetup -y --use-random luksFormat /dev/nvme0n1p6 # (Root partition)
cryptsetup luksOpen /dev/nvme0n1p6 cryptroot

mkfs.ext4 /dev/nvme0n1p5 # (Boot Partition)
mkfs.ext4 /dev/mapper/cryptroot
```

### Mount partitions, pacstrap, fstab
```bash
mount /dev/mapper/cryptroot /mnt
mkdir /mnt/boot
mount /dev/nvme0n1p5 /mnt/boot # Mount boot partition
mkdir /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi # Mount EFI partition

pacstrap /mnt linux linux-headers linux-firmware base base-devel git nano mesa intel-media-driver libinput pipewire wireplumber pacman-contrib reflector networkmanager curl

genfstab genfstab -U /mnt >> /mnt/etc/fstab
```

### `arch-chroot /mnt` To enter the system

### Swapfile
```bash
fallocate -l 4GB /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

nano /etc/fstab
# ADD:
/swapfile none swap sw 0 0
```

### Initial setup
```bash
ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
hwclock --systohc
nano /etc/locale.gen # Uncomment en_US.UTF-8 UTF-8 
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "Installation00" >> /etc/hostname
nano /etc/hosts
# 127.0.0.1    localhost
# ::1          localhost
# 127.0.1.1    Installation00.localdomain Installation00
```

### Ramdisk
```bash
nano /etc/mkinitcpio.conf
# Make the line with HOOKS as below
HOOKS=(base udev autodetect keyboard modconf block encrypt filesystems fsck)
# Save and exit nano
mkinitcpio -p linux
```

### Initial user administration
```bash
passwd # Set root password
useradd -m -G wheel abhay
passwd abhay
EDITOR=nano visudo
# Uncomment %wheel ALL=(ALL:ALL) ALL
```

### `systemctl enable NetworkManager.service` Enable NetworkManager

### Grub
```bash
pacman -S grub efibootmgr os-prober
# Run os-prober and check output

nano /etc/default/grub
# GRUB_CMDLINE_LINUX="cryptdevice=UUID={ROOT_UUID_FROM_blkid}:cryptroot root=/dev/mapper/cryptroot"
# Uncomment line to allow os-prober to run

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB 
grub-mkconfig -o /boot/grub/grub.cfg 
```

### Exit and reboot
```bash
exit
umount -R /mnt
reboot
```

## After Installation

### Connect to internet with NetworkManager
```bash
nmcli device wifi list
nmcli device wifi connect <SSID> password <password>
nmcli connection show # connection info
```

### Install yay
```bash
git clone https://aur.archlinux.org/yay-bin.git # Change yay-bin to yay if you want to compile yourself
cd yay-bin
makepkg -si
```

### Fun Stuff
```bash
sudo pacman -S neofetch cmatrix
yay -S pipes.sh

# btop (The unofficial pcakage in the community repo is broken)
mkdir btopfiles # Maybe in Downloads
curl -L https://github.com/aristocratos/btop/releases/download/v1.2.8/btop-x86_64-linux-musl.tbz
tar -x -j -f "btop-x86_64-linux-musl.tbz" -C btopfiles
cd btopfiles
./install.sh
```
