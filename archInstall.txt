#!/bin/bash

set -e

DRIVE="$1"

dd if=/dev/zero of=$DRIVE bs=1M count=10 status=progress
printf "g\nn\n1\n\n+512M\nt\n1\nn\n2\n\n\nw\n" | fdisk "$DRIVE"
partprobe "$DRIVE"
mkfs.fat -F32 "${DRIVE}1"
mkfs.ext4 "${DRIVE}2"
mount "${DRIVE}2" /mnt
mkdir /mnt/boot
mount "${DRIVE}1" /mnt/boot
reflector --latest 5 --country US --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
pacstrap -K /mnt base linux linux-firmware networkmanager intel-ucode fasm clang neovim git doas zsh
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt /bin/bash << EOF
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
hwclock --systohc
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "curry" > /etc/hostname
systemctl enable NetworkManager
echo "root:mango" | chpasswd
useradd -m -G wheel,audio,video,optical,storage,input curry
echo "curry:mango" | chpasswd
echo "permit persist :wheel" > /etc/doas.conf
chmod 0400 /etc/doas.conf
pacman -S grub efibootmgr os-prober nvidia nvidia-utils nvidia-settings --noconfirm
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
pacman -S man-db man-pages reflector ghostty chromium fastfetch ripgrep fzf bat zip unzip bluez bluez-utils bluez-deprecated-tools xorg-server xorg-xinit xfce4 xfce4-goodies lightdm lightdm-gtk-greeter --noconfirm
sed -i 's/^MODULES=.*/MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)/' /etc/mkinitcpio.conf
mkinitcpio -P
echo "options nvidia-drm modeset=1" > /etc/modprobe.d/nvidia.conf
mkdir -p /home/curry/.config
chown -R curry:curry /home/curry
timedatectl set-ntp true
systemctl enable bluetooth
systemctl enable lightdm
umount -R /mnt
reboot
EOF