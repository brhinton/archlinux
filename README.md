# archlinux
Arch Linux

```
disk=blkdev
efipart=blkdevprt1
linuxpart=blkdevprt2
hostname=lc
ucode=intel-ucode
timedatectl set-ntp true
wipefs --all /dev/${disk}
sgdisk /dev/${disk} -n 1::+512MiB -t 1:EF00
sgdisk /dev/${disk} -n 2 -t 2:8300
cryptsetup luksFormat --cipher aes-xts-plain64 --keyslot-cipher serpent-xts-plain --keyslot-key-size 512 --use-random -S 0 -h sha512 -i 4000 /dev/${linuxpart}
cryptsetup open /dev/${linuxpart} root
mkfs.ext4 /dev/mapper/root
mount /dev/mapper/root /mnt
mkfs.fat -F32 /dev/${efipart}
mount --mkdir /dev/${efipart} /mnt/efi
dd if=/dev/zero of=/mnt/swapfile bs=1M count=8000 status=progress
chmod 600 /mnt/swapfile
mkswap /mnt/swapfile
swapon /mnt/swapfile
pacstrap -K /mnt base base-devel linux linux-hardened linux-hardened-headers linux-firmware apparmor mesa xf86-video-intel vulkan-intel git vi vim ukify
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
ln -sf /usr/share/zoneinfo/UTC /etc/localtime
hwclock --systohc
sed -i 's/#'"en_US.UTF-8"' UTF-8/'"en_US.UTF-8"' UTF-8/g' /etc/locale.gen
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
echo ${hostname} > /etc/hostname
cat <<EOT >> /etc/hosts
127.0.0.1 ${hostname}
::1 localhost
127.0.1.1 ${hostname}.localdomain ${hostname}
EOT
sed -i 's/HOOKS.*/HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems resume fsck)/' /etc/mkinitcpio.conf
mkinitcpio -P
bootctl install
passwd
pacman -S ${ucode} efibootmgr
swapoffset=`filefrag -v /swapfile | awk '/\s+0:/ {print $4}' | sed -e 's/\.\.$//'`
linuxpartuuid=$(blkid -s UUID -o value /dev/${linuxpart})
efibootmgr --disk /dev/${efipart} --part 1 --create --label "Linux" --loader /vmlinuz-linux --unicode "cryptdevice=UUID=$linuxpartuuid:root root=/dev/mapper/root resume=/dev/mapper/root resume_offset=$swapoffset rw initrd=\intel-ucode.img initrd=\initramfs-linux.img" --verbose
cat <<EOT >> /etc/mkinitcpio.d/linux.preset
ALL_kver="/boot/vmlinuz-linux"
ALL_microcode=(/boot/*-ucode.img)
PRESETS=('default' 'fallback')
default_uki="/efi/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
EOT
mkdir -p /efi/EFI/Linux
cat <<EOT >> /etc/kernel/cmdline
cryptdevice=UUID=<uuid of root cryptdevice>:root root=/dev/mapper/root resume=/dev/mapper/root resume_offset=51347456 rw
EOT
mkinitcpio -p linux
echo "layout=uki" >> /etc/kernel/install.conf
cat <<EOT >> /etc/systemd/network/nic0.network
[Match]
Name=nic0
[Network]
DHCP=yes
EOT
pacman -Syu
pacman -S xorg xfce4 xfce4-goodies lightdm lightdm-gtk-greeter libva-intel-driver mesa xorg-server xorg-xinit 
systemctl enable lightdm
systemctl start lightdm
systemctl enable systemd-resolved.service
systemctl enable systemd-networkd.service
systemctl start systemd-resolved.service
systemctl start systemd-networkd.service
useradd -m -g wheel -s /bin/bash dev
pacman -S sudo
sed -i 's/# %wheel ALL=(ALL:ALL) NOPASSWD: ALL/%wheel ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
passwd dev
```



