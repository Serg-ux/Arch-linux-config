# Arch Linux Installation Guide
## Prerequisites
First of all, ensure you have an Arch Linux image on a USB drive or disk, and it must be bootable.

## Step 1: Check Network Connectivity
To verify if your network interface is active, use the following command:
```bash
ip addr show
```

In my case, I’m using a laptop, and I want to connect via Wi-Fi, so let's proceed with configuring the Wi-Fi.

Note: You can also use an Ethernet cable to skip this step.

## Step 2: Set Up Wi-Fi Connection
To configure Wi-Fi, run the following command to access the Wi-Fi configuration prompt:
```bash
iwctl
```
To list available Wi-Fi networks:
```bash
station wlan0 get-networks
```
Exit the prompt with:
```bash
exit
```
Now, to connect to your Wi-Fi network, use the following command, replacing "*******" with your Wi-Fi password and NAME_OF_NETWORK with the actual network name:
```bash
iwctl --passphrase "*******" station wlan0 connect NAME_OF_NETWORK
```
To verify if you have an IP address, run:

```bash
ip addr show
```

## Step 3: Partitioning the Disk
Let’s list all the storage volumes available with:

```bash
lsblk
```

To create partitions on the disk, use fdisk:
```bash
fdisk /dev/nvme0n1
```

Create a new empty partition table using:

```bash
g
```

Now, create a new partition by entering:

```bash
n
```
Next, create a second partition.

Afterward, you can check the partitions created:
```bash
lsblk
```
For the remaining free space, create a third partition.

Now, assign the partition type with:
```bash
t
```
Set the partition type to LVM for partition 3.

Ensure you have all three partitions as intended:
```bash
lsblk
```
Write the partition changes with:
```bash
w
```
Warning: This action will erase all data on the disk.

## Step 4: Format Partitions
To format the first partition as FAT32:
```bash
mkfs.fat -F32 /dev/nvme0n1p1
```
To format the second partition as EXT4:
```bash
mkfs.ext4 /dev/nvme0n1p2
```
The third partition will be encrypted in the next step.

## Step 5: Set Up an Encrypted Partition
We will now encrypt the third partition:
```bash
cryptsetup luksFormat /dev/nvme0n1p3
```
Open the encrypted partition:

```bash
cryptsetup open --type luks /dev/nvme0n1p3 lvm
```
## Step 6: Configure LVM
Create the physical volume:
```bash
pvcreate /dev/mapper/lvm
```

Create a volume group:
```bash
vgcreate volgroup0 /dev/mapper/lvm
```
Create logical volumes:
```bash
lvcreate -L 30GB volgroup0 -n lv_root
```
```bash
lvcreate -L 250GB volgroup0 -n lv_home
```
To view the logical volumes:


lvdisplay
Enable the required kernel modules:
```bash
modprobe dm_mod
```
```bash
vgscan
```
```bash
vgchange -ay
```

Format the logical volumes as EXT4:
```bash
mkfs.ext4 /dev/volgroup0/lv_root
```
```bash
mkfs.ext4 /dev/volgroup0/lv_home
```
Now, mount the partitions:

```bash
mount /dev/volgroup0/lv_root /mnt
```
```bash
mkdir /mnt/boot
```
```bash
mount /dev/nvme0n1p2 /mnt/boot
```
```bash
mkdir /mnt/home
```
```bash
mount /dev/volgroup0/lv_home /mnt/home
```

## Step 7: Install Required Packages
Install the base system packages:
```bash
pacstrap -i /mnt base
```
## Step 8: Generate the fstab File
To generate the fstab file for mounting partitions:
```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
```
```bash
cat /mnt/etc/fstab
```
## Step 9: Chroot into the New System
Chroot into the installed system:
```bash
arch-chroot /mnt
```

## Step 10: Set Up Users
Set the root password:
```bash
passwd
```


Create a user:

```bash
useradd -m -g users -G wheel $USER
```
```bash
passwd $USER
```

## Step 11: Install Additional Packages
Install necessary utilities and packages:
```bash
pacman -S base-devel dosfstools grub efibootmgr gnome gnome-tweaks lvm2 mtools nano networkmanager openssh os-prober sudo
```
## Step 12: Enable SSH
Enable the SSH service:

```bash
systemctl enable sshd
```

## Step 13: Install the Linux Kernel
Install the Linux kernel and headers:
```bash
pacman -S linux linux-headers linux-lts linux-lts-headers
```
```bash
pacman -S linux-firmware
```


## Step 14: Install GPU Drivers
To install GPU drivers, identify your GPU using:

```bash
lspci
```

For AMD GPUs:
```bash
pacman -S mesa
```

For Intel GPUs:
```bash
pacman -S intel-media-driver
```

For NVIDIA GPUs:
```bash
pacman -S nvidia nvidia-utils nvidia-lts
```

## Step 15: Generate RAM Disks for the Kernel
Edit the mkinitcpio.conf file to add encrypt and lvm2:
```bash
nano /etc/mkinitcpio.conf
```


Generate the initramfs images for your kernels:
```bash
mkinitcpio -p linux
```
```bash
mkinitcpio -p linux-lts
```
## Step 16: Configure Locale and GRUB
Configure the locale:
```bash
nano /etc/locale.gen
```
Uncomment your desired locale, then generate the locale:

```bash
locale-gen
```

Edit the GRUB configuration:

```bash
nano /etc/default/grub
```

Create the EFI directory and mount the first partition:

```bash
mkdir /boot/EFI
```
```bash
mount /dev/nvme0n1p1 /boot/EFI
```

Install the GRUB bootloader:

```bash
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
```

Copy the GRUB locale files:

```bash
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```

Generate the GRUB configuration file:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Enable GDM and NetworkManager:
```bash
systemctl enable gdm
```
```bash
systemctl enable NetworkManager
```
## Step 17: Final Steps
Exit the chroot environment:

```bash
exit
```
Unmount all partitions:

```bash
umount -a
```
Reboot the system:

```bash
reboot
