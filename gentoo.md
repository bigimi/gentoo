#  disk EFI partition
mkfs.vfat -F 32 /dev/nvme0n1p1
mkdir -p /mnt/gentoo/efi
mount /dev/nvme0n1p1 /mnt/gentoo/efi

# disk swap partition
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2

# disk root partition
mkfs.btrfs /dev/nvme0n1p3
mkdir -p /mnt/gentoo
mount /dev/nvme0n1p3 /mnt/gentoo

# create BTRFS subvolumes
btrfs subvolume create /mnt/gentoo/@
btrfs subvolume create /mnt/gentoo/@home
btrfs subvolume create /mnt/gentoo/@.snapshots

# unmount /mnt/gentoo
umount /mnt/gentoo

# mount subvolumes (compression is optional)
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p3 /mnt/gentoo

mkdir -p /mnt/gentoo/home
mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p3 /mnt/gentoo/home

mkdir -p /mnt/gentoo/.snapshots
mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p3 /mnt/gentoo/.snapshots

###########################################################################
# stage 3
cd /mnt/gentoo

# time sync
chronyd -q

# download stage 3
links https://www.gentoo.org/downloads/mirrors/

# unpack stage 3
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner

# config compile opts
nano /mnt/gentoo/etc/portage/make.conf


                 file setup


##############################################################################
# cp network
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/

# source profile
source /etc/profile

# chroot
arch-chroot /mnt/gentoo
export PS1="(chroot) ${PS1}"

# Sync the ebuild repository for the latest packages
emerge --sync

# Read news items
eselect news list
eselect news read

# select correct profile
eselect profile list | less 
eselect profile set 1

# set gentoo.conf
nano  /etc/portage/binrepos.conf/gentoo.conf

[binhost]
priority = 9999
sync-uri = https://distfiles.gentoo.org/releases/amd64/binpackages/23.0/x86-64-v3

#  keyring for verification
getuto

# accept license
nano /etc/portage/package.use/system

                     system file setup

# update world set (bynary packages)
emerge --ask --verbose --update --deep --newuse --getbinpkg @world

# remove obsoleted packages
emerge --ask --depclean

###########################################################################
# timezone
ln -sf ../usr/share/zoneinfo/Europe/London  /etc/localtime

# config locales
nano /etc/locale.gen

# generate locales
locale-gen

# select locale 
eselect locale list
eselect locale set  4

# update env
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"

##############################################################################
# install firmware
emerge --ask sys-kernel/linux-firmware  sys-firmware/sof-firmware

# efi stub
**emerge --ask sys-kernel/installkernel**

mkdir -p /efi/EFI/Gentoo
nano /etc/default/efi/EFI/efi-mkconfig

                file setup

# install binary kernel
emerge -a sys-kernel/gentoo-kernel-bin

#################################################################################

# fstab
genfstab -U / >> /etc/fstab

# check fstab
nano /etc/fstab

# networking
echo gentoo > /etc/hostname (openrc hostname.d.....)

# hosts config
nano /etc/hosts

    127.0.0.1     gentoo localhost
    ::1                gentoo localhost

# dhcpcd & NetworkManager
emerge --ask net-misc/dhcpcd  net-misc/networkmanager

rc-update add dhcpcd default
rc-update add NetworkManager default

# system tools
emerge-a sys-apps/mlocate app-shells/bash-completion net-misc/chrony \
 sys-fs/btrfs-progs sys-block/io-scheduler-udev-rules 
rc-update add chronyd default
rc-update add sshd default

# sysklog
emerge --ask app-admin/sysklogd
rc-update add sysklogd default

# crony
emerge --ask sys-process/cronie
rc-update add cronie default

# doas
emerge --ask app-admin/doas
nano /etc/doas.conf

    permit persist :wheel
	permit nopass bigimi cmd /sbin/reboot


#########################################################################
# root passwd
passwd

# add user
useradd -m -G users,wheel,audio,video,usb -s /bin/bash bigimi

# reboot
exit
umount -R /mnt/gentoo
reboot
 
 
 *********** E N D ***********



