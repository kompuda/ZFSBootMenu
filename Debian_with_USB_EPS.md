
# DEBIAN ON ZFSBOOTMENU
  
## BOOT FROM DEBIAN LIVE INSTALLER
Username = user  Password = live
    
### IN THE TERMINAL BECOME ROOT
```
sudo -i
```
  
### SOURCE ```/etc/os-release```
```
source /etc/os-release
export ID
```
  
### CONFIGURE AND UPDATE APT
```
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib
EOF
apt update
```
  
### INSTALL ZFS, DEBOOTSTRAP AND HELPERS INTO THE LIVE DEBIAN ENVIRONMENT 
```
apt install debootstrap gdisk dkms linux-headers-$(uname -r)
apt install zfsutils-linux
```
  
### GENERATE ```/etc/hostid```
```
zgenhostid -f 0x00bab10c
```

## DEFINE DISK VARIABLES (NVME OR SEPARATE USB FLASH)
### SEPARATE USB BOOT DEVICE
```
export BOOT_DISK="/dev/sdb"
export BOOT_PART="1"
export BOOT_DEVICE="${BOOT_DISK}${BOOT_PART}"
```
Then define variables that refer to the disk and partition number that will hold the ZFS pool
```
export POOL_DISK="/dev/nvme0n1"
export POOL_PART="2"
export POOL_DEVICE="${POOL_DISK}p${POOL_PART}"
```


## DISK PREPARATION
  
### WIPE NVME PARTITIONS
```
zpool labelclear -f "$POOL_DISK"

wipefs -a "$POOL_DISK"

sgdisk --zap-all "$POOL_DISK"
```
  
 
### CREATE ZPOOL PARTITION
```
sgdisk -n "${POOL_PART}:0:-10m" -t "${POOL_PART}:bf00" "$POOL_DISK"
```

## ZFS POOL CREATION

### CREATE THE POOL - UNENCRYPTED
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
  
### CREATE INITIAL FILE SYSTEMS
``` 
zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/${ID}
zfs create -o mountpoint=/home zroot/home

zpool set bootfs=zroot/ROOT/${ID} zroot
```
  
### EXPORT, THEN RE-IMPORT WITH TEMP MOUNTPOINT - UNENCRYPTED
```
zpool export zroot
```
```
zpool import -N -R /mnt zroot
zfs mount zroot/ROOT/${ID}
zfs mount zroot/home
```

### VERIFY EVERYTHING MOUNTS CORRECTLY
It should look like the two zroot lines below the mount command
```
mount | grep mnt
```
zroot/ROOT/debian on /mnt type zfs (rw,relatime,xattr,posixacl)

zroot/home on /mnt/home type zfs (rw,relatime,xattr,posixacl)

### UPDATE DEVICE SYMLINKS
```
udevadm trigger
```
  
### INSTALL DEBOOTSTRAP DEBIAN ONTO YOUR DISK
```
debootstrap --arch=amd64 trixie /mnt
```
  
### COPY FILES INTO THE NEW DEBOOTSTRAP DEBIAN INSTALL
```
cp /etc/hostid /mnt/etc
cp /etc/resolv.conf /mnt/etc
```

### CHROOT INTO THE NEW DEBOOTSTRAP DEBIAN OS
```
#This version handles nested mounts automatically.
mount --rbind /dev  /mnt/dev
mount --rbind /proc /mnt/proc
mount --rbind /sys  /mnt/sys
chroot /mnt /bin/bash
```


# BASIC SYSTEM CONFIGURATION


### CONFIGURE THE HOSTNAME
Replace `HOSTNAME' with the desired hostname
```
echo 'YOURHOSTNAME' > /etc/hostname
```
```
nano /mnt/etc/hosts

#Add a line:
127.0.1.1   YOURHOSTNAME 
#or if the system has a real name in DNS:
127.0.1.1       FQDN HOSTNAME
```

### CONFIGURE THE NETWORK INTERFACE
```
# Find the interface name
ip addr show
```
Adjust NAME below to match your interface name
```
nano /mnt/etc/network/interfaces.d/NAME

#Add lines to config:
auto NAME
iface NAME inet dhcp
```
  
### SET A ROOT PASSWORD
```
passwd root
```
  
### CONFIGURE APT SOURCES
```
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware non-free contrib
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware non-free contrib

deb http://deb.debian.org/debian-security trixie-security main non-free-firmware non-free contrib
deb-src http://deb.debian.org/debian-security/ trixie-security main non-free-firmware non-free contrib

# trixie-updates, to get updates before a point release is made;
deb http://deb.debian.org/debian trixie-updates main non-free-firmware non-free contrib
deb-src http://deb.debian.org/debian trixie-updates main non-free-firmware non-free contrib
EOF
```

### UPDATE REPO CACHE
```
apt update
```

### UPGRADE THE MINIMAL DEBOOTSTRAP DEBIAN
```
apt dist-upgrade -y
```

### INSTALL BASE PACKAGES
```
apt install linux-headers-amd64 linux-image-amd64 zfsutils-linux zfs-initramfs zfs-dkms -y
```
```
apt install git aptitude firmware-linux wireless-tools firmware-realtek kexec-tools systemd-timesyncd tasksel network-manager bind9-dnsutils pciutils usbutils -y
```
```
apt install dosfstools dpkg-dev curl nano -y
```

### INSTALL LOCALISATION PACKAGES
```
apt install locales keyboard-configuration console-setup
```

### CONFIGURE PACKAGES TO CUSTOMISE LOCALES AND CONSOLE PROPERTIES
```
dpkg-reconfigure locales tzdata keyboard-configuration console-setup
```


## ZFS CONFIGURATION

```
echo "REMAKE_INITRD=yes" > /etc/dkms/zfs.conf
```

### ENABLE SYSTEMD ZFS SERVICES
```
systemctl enable zfs.target
systemctl enable zfs-import-cache.service
systemctl enable zfs-mount.service
systemctl enable zfs-import.target
```

### CONFIGURE ```initramfs-tools```
n/a
  
### REBUILD THE INITRAMFS
```
update-initramfs -c -k all
```

### DOUBLE CHECK 
```
zpool set bootfs=zroot/ROOT/debian zroot
update-initramfs -c -k all
```

### ZFSBootMenu DRY RUN BEFORE REBOOTING
```
zpool get bootfs zroot
```
Expected output
NAME   PROPERTY  VALUE                SOURCE
zroot  bootfs    zroot/ROOT/debian    local

If it doesn't show zroot/ROOT/debian, fix it with:
```
zpool set bootfs=zroot/ROOT/debian zroot
```
That’s what ZFSBootMenu uses to pick the default boot dataset.

### CONFIRM YOUR DATASETS LOOK RIGHT
```
zfs list -o name,canmount,mountpoint
```

Expected:
NAME               CANMOUNT  MOUNTPOINT
zroot              off       none
zroot/ROOT         off       none
zroot/ROOT/debian  noauto    /
zroot/home         on        /home

### CHECK KERNEL AND INITRAMFS PRESENCE
```
/boot/vmlinuz-<version>
/boot/initrd.img-<version>
```
So list them:
```
ls -lh /mnt/boot
```
You should see files like:
vmlinuz-6.12.43-amd64
initrd.img-6.12.43-amd64

If you’re in a chroot, remove /mnt:
```
ls -lh /boot
```

### OPTIONAL DRY-RUN USING ZFSBOOTMENU TOOLS
If you’ve installed the package inside your chroot:
```
apt install zfsbootmenu
```
Then you can simulate what ZFSBootMenu will detect:
```
zfsbootmenu --print
```
It’ll show something like:
Discovered boot environments:
  zroot/ROOT/debian

If it lists your zroot/ROOT/debian dataset and finds the kernel and initramfs under /boot, you’re golden.


### SANITY REBOOT PLAN

When you reboot:

Plug in your ZFSBootMenu USB.

Select it in your UEFI boot menu.

It should open ZFSBootMenu, show zroot/ROOT/debian, and boot into it automatically.

If you see your dataset but it won’t boot, it’s almost always because of:

Missing initramfs ZFS modules (fixed by update-initramfs), or

Wrong bootfs (fixed above).


# ################################################################################



## PREPARE FOR FIRST BOOT

### EXIT THE CHROOT, UNMOUNT EVERYTHING
```
exit
```
```
umount -n -R /mnt
```

### EXPORT THE ZPOOL AND REBOOT
```
zpool export zroot
reboot
```


## AFTER FIRST BOOT

### CREATE A USER ACCOUNT

username=YOUR_USERNAME
```
sudo zfs create rpool/home/$username
adduser $username

cp -a /etc/skel/. /home/$username
chown -R $username:$username /home/$username
usermod -a -G audio,cdrom,dip,floppy,netdev,plugdev,sudo,video $username
```

### INSTALL A REGULAR SET OF DEBIAN SOFTWARE & DESKTOP ENVIRONMENT
```
tasksel --new-install
```

### INSTALL SOFTWARE
```
sudo apt install synaptic wget fastfetch gcc make python3 python3-pip unrar unzip cargo p7zip ntfs-3g htop ffmpeg ttf-mscorefonts-installer fonts-firacode fonts-jetbrains-mono fonts-croscore fonts-crosextra-carlito fonts-crosextra-caladea fonts-noto fonts-noto-cjk -y
```

### DISABLE LOG COMPRESSION
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

### REBOOT YOUR MACHINE
```
reboot
```

# FINAL CLEANUP

## DISABLE root SSH LOGINS
```
sudo vi /etc/ssh/sshd_config
# Remove: PermitRootLogin yes

sudo systemctl restart ssh
```

## RE-ENABLE GRAPHICAL BOOT PROCESS
```
sudo vi /etc/default/grub
# Add quiet to GRUB_CMDLINE_LINUX_DEFAULT
# Comment out GRUB_TERMINAL=console
# Save and quit.

sudo update-grub
```

## VERIFY USB EFI PARTITION
```
lsblk -f
```
## CONFIRM EFI STRUCTURE ON THE USB
```
sudo tree /mnt/boot/efi
#or if not mounted yet
sudo mount /dev/sdb1 /mnt
sudo tree /mnt
```
## CREATE A UEFI BOOT ENTRY
```
sudo efibootmgr --create \
  --disk /dev/sdb \
  --part 1 \
  --label "ZFSBootMenu USB" \
  --loader '\EFI\BOOT\BOOTX64.EFI'
```
## VERIFY THE NEW BOOT ENTRY
```
sudo efibootmgr -v
```
## ADJUST BOOT ORDER
```
sudo efibootmgr -o 0004,0001,0000
# replacing 0004 with the actual Boot#### number shown for "ZFSBootMenu USB"
```








# #######################################################################################

## FOR ZFSBOOTMENU ON DISK

# INSTALL AND CONFIGURE ZFSBOOTMENU

### SET ZFSBOOTMENU PROPERTIES ON DATASETS
```
zfs set org.zfsbootmenu:commandline="quiet" zroot/ROOT
```

### CREATE A VFAT FILESYSTEM
```
mkfs.vfat -F32 "$BOOT_DEVICE"
```

### CREATE AN FSTAB ENTRY AND MOUNT
```
cat << EOF >> /etc/fstab
$( blkid | grep "$BOOT_DEVICE" | cut -d ' ' -f 2 ) /boot/efi vfat defaults 0 0
EOF

mkdir -p /boot/efi
mount /boot/efi
```

### INSTALL ZFSBOOTMENU
Fetch a prebuilt EFI executable and save it into the EFI system partition.
```
mkdir -p /boot/efi/EFI/ZBM
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI
```

### CONFIGURE EFI BOOT ENTRIES
```
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

DIRECT USING EFI
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



