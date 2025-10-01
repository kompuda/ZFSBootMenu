
# DEBIAN ON ZFSBOOTMENU

## BOOT FROM DEBIAN LIVE INSTALLER
Username = user  Password = live

### In the Terminal Become root
```
sudo -i
```

### Source /etc/os-release
```
source /etc/os-release
export ID
```
### Configure and Update APT
```
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib
EOF
apt update
```
### Install ZFS, Debootstrap and Helpers into the Live Debian Environment 
```
apt install debootstrap gdisk dkms linux-headers-$(uname -r)
apt install zfsutils-linux
```
### Generate /etc/hostid
```
zgenhostid -f 0x00bab10c
```

## DEFINE DISK VARIABLES
### Single NVME Disk
```
export BOOT_DISK="/dev/nvme0n1"
export BOOT_PART="1"
export BOOT_DEVICE="${BOOT_DISK}p${BOOT_PART}"

export POOL_DISK="/dev/nvme0n1"
export POOL_PART="2"
export POOL_DEVICE="${POOL_DISK}p${POOL_PART}"
```

## DISK PREPARATION

### Wipe partitions
```
zpool labelclear -f "$POOL_DISK"

wipefs -a "$POOL_DISK"
wipefs -a "$BOOT_DISK"

sgdisk --zap-all "$POOL_DISK"
sgdisk --zap-all "$BOOT_DISK"
```

### Create EFI boot partition
```
sgdisk -n "${BOOT_PART}:1m:+512m" -t "${BOOT_PART}:ef00" "$BOOT_DISK"
```
### Create ZPOOL partition
```
sgdisk -n "${POOL_PART}:0:-10m" -t "${POOL_PART}:bf00" "$POOL_DISK"
```

## ZFS POOL CREATION

### Create the Pool - Unencrypted
```
zpool create -f -o ashift=12 \
 -O compression=lz4 \
 -O acltype=posixacl \
 -O xattr=sa \
 -O relatime=on \
 -o autotrim=on \
 -o compatibility=openzfs-2.2-linux \
 -m none zroot "$POOL_DEVICE"
```

### Create Initial File Systems
``` 
zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/${ID}
zfs create -o mountpoint=/home zroot/home

zpool set bootfs=zroot/ROOT/${ID} zroot
```

### Export, then re-import with temp mountpoint - Unencrypted
```
zpool export zroot
zpool import -N -R /mnt zroot
zfs mount zroot/ROOT/${ID}
zfs mount zroot/home
```

### Verify everything mounts correctly
It should look like the two zroot lines below the mount command
```
mount | grep mnt
```
zroot/ROOT/debian on /mnt type zfs (rw,relatime,xattr,posixacl)   
zroot/home on /mnt/home type zfs (rw,relatime,xattr,posixacl)

### Update Device Symlinks
```
udevadm trigger
```
### Install Debootstrap Debian onto your disk
```
debootstrap trixie /mnt
```
### Copy Files into the new Debootstrap Debian Install
```
cp /etc/hostid /mnt/etc
cp /etc/resolv.conf /mnt/etc
```

### Chroot into the new Debootstrap Debian OS
```
mount -t proc proc /mnt/proc
mount -t sysfs sys /mnt/sys
mount -B /dev /mnt/dev
mount -t devpts pts /mnt/dev/pts
chroot /mnt /bin/bash
```


## BASIC DEBIAN CONFIGURATION


### Set a Hostmane
```
echo 'YOURHOSTNAME' > /etc/hostname
echo -e '127.0.1.1\tYOURHOSTNAME' >> /etc/hosts
```
### Set a root password
```
echo 'YOURHOSTNAME' > /etc/hostname
echo -e '127.0.1.1\tYOURHOSTNAME' >> /etc/hosts
```
### Configure APT Sources
```
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib

deb http://deb.debian.org/debian-security trixie-security main non-free-firmware contrib
deb-src http://deb.debian.org/debian-security/ trixie-security main non-free-firmware contrib

# trixie-updates, to get updates before a point release is made;
deb http://deb.debian.org/debian trixie-updates main non-free-firmware contrib
deb-src http://deb.debian.org/debian trixie-updates main non-free-firmware contrib
EOF
```
### Update repo cache
```
apt update
```
### Install base packages
```
apt install linux-headers-amd64 linux-image-amd64 zfsutils-linux zfs-initramfs zfs-dkms -y
```
```
apt install firmware-linux wireless-tools firmware-realtek systemd-boot kexec-tools systemd-timesyncd network-manager bind9-dnsutils pciutils usbutils -y
```
```
apt install dosfstools dpkg-dev curl nano -y
```

### Install additonal base packages
```
apt install locales keyboard-configuration console-setup
```
### Configure packages to customise locales and console properties
```
dpkg-reconfigure locales tzdata keyboard-configuration console-setup
```


## ZFS CONFIGURATION

### 
```
echo "REMAKE_INITRD=yes" > /etc/dkms/zfs.conf
```

### Enable systemd ZFS services
```
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target
```

### Configure initramfs-tools
n/a

### Rebuild the initramfs
```
update-initramfs -c -k all
```


## INSTALL AND CONFIGURE ZFSBOOTMENU

### Set ZFSBootMenu properties on datasets
```
zfs set org.zfsbootmenu:commandline="quiet" zroot/ROOT
```

### Create a vfat filesystem
```
mkfs.vfat -F32 "$BOOT_DEVICE"
```

### Create an fstab entry and mount
```
cat << EOF >> /etc/fstab
$( blkid | grep "$BOOT_DEVICE" | cut -d ' ' -f 2 ) /boot/efi vfat defaults 0 0
EOF

mkdir -p /boot/efi
mount /boot/efi
```

### Install ZFSBootMenu
Fetch a prebuilt EFI executable and save it into the EFI system partition.
```
mkdir -p /boot/efi/EFI/ZBM
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI
```

### Configure EFI boot entries
```
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```
Direct using EFI
```
apt install efibootmgr
```
```
efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
  -L "ZFSBootMenu (Backup)" \
  -l '\EFI\ZBM\VMLINUZ-BACKUP.EFI'

efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
  -L "ZFSBootMenu" \
  -l '\EFI\ZBM\VMLINUZ.EFI'
```


## PREPARE FOR FIRST BOOT

### Exit the chroot, unmount everything
```
exit
```
```
umount -n -R /mnt
```

### Export the zpool and reboot
```
zpool export zroot
reboot
```


## AFTER FIRST BOOT

### Create a user account
username=YOUR_USERNAME
```
zfs create rpool/home/$username
adduser $username

cp -a /etc/skel/. /home/$username
chown -R $username:$username /home/$username
usermod -a -G audio,cdrom,dip,floppy,netdev,plugdev,sudo,video $username
```

### Upgrade the minimal debootstrap Debian
```
apt update
apt dist-upgrade -y
```

### Install a regular set of Debian software
```
tasksel --new-install
```

### Install software
```
sudo apt install git aptitude synaptic wget fastfetch gcc make python3 python3-pip unrar unzip cargo p7zip ntfs-3g htop ffmpeg ttf-mscorefonts-installer fonts-firacode fonts-jetbrains-mono fonts-croscore fonts-crosextra-carlito fonts-crosextra-caladea fonts-noto fonts-noto-cjk -y
```

### Disable log compression
```
nano /etc/logrotate.d
```
```
for file in /etc/logrotate.d/* ; do
    if grep -Eq "(^|[^#y])compress" "$file" ; then
        sed -i -r "s/(^|[^#y])(compress)/\1#\2/" "$file"
    fi
done
```

### Reboot your machine.
