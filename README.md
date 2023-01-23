# Installation steps
Replace \<VAR\> with appropriate value.

## Image
### Download image
https://artixlinux.org/download.php

### Verify image
gpg --keyserver pgpkeys.eu --recv-keys <KEY>
gpg --auto-key-retrieve --verify <SIG_FILE> <ISO_FILE>

### Burn
sudo dd if=<ISO_FILE> iflag=nocache of=/dev/<USB> oflag=direct status=progress
sync

## Installation
### Connect to wifi
connmanctl
scan wifi
services
agent on
connect wifi_code <CODE>
ping 1.1.1.1 # test

### Enable ssh
ln -s /etc/runit/sv/sshd /run/runit/service/

### Connect using ssh
ip a # find ip on machine 1
ssh artix@<IP> # connect on machine 2

### Root
su

### Keyboard
ls /usr/share/kbd/keymaps/i386/qwerty/
loadkeys us

### Locate drive
lsblk
fdisk -l

### Random write to drive
dd if=/dev/urandom of=/dev/sdX

### Parition drive
cfdisk /dev/sdX
gpt partition table
3G for boot (D1)
rest for root (D2)

### Encrypt
cryptsetup luksFormat --type luks1 /dev/(D2)
cryptsetup open /dev/(D2) cryptlvm (any name)
pvcreate /dev/mapper/cryptlvm
vgcreate vg0 (any name) /dev/mapper/cryptlvm
lvcreate -l +100%FREE vg0 -n root
mkfs.ext4 -L root /dev/mapper/vg0-root
mkfs.vfat -F32  /dev/(D1)

### Mount
mount /dev/(D1) /mnt/efi/
mount /dev/mapper/vg0-root /mnt/

### pacman
vi /etc/pacman.d/mirrorlist # put close mirrors on top

### strap
basestrap /mnt base base-devel runit elogind-runit grub linux linux-firmware intel-ucode efibootmgr lvm2 mkinitcpio vi cryptsetup cryptsetup-runit lvm2-runit git

### fstab
= fstab =
fstabgen -U /mnt >> /mnt/etc/fstab

### chroot
artix-chroot /mnt

### Basic setup
1. vi /etc/vconsole.conf
FONT=lat5-16
KEYMAP=us-acentos
1. ln -sf /usr/share/zoneinfo/<ZONE> /etc/localtime
2. hwclock --systohc
3. vi /etc/locale.gen, uncomment en_US.UTF-8 UTF-8
4. locale-gen
5. echo LANG=en_US.UTF-8 > /etc/locale.conf
6. echo my-hostname > /etc/hostname

### Keyfile
dd bs=512 count=4 if=/dev/urandom of=<CRYPT_FILE>
chmod 000 <CRYPT_FILE>
cryptsetup luksAddKey /dev/(D2) <CRYPT_FILE>

in /etc/mkinitcpio.conf
FILES=(<CRYPT_FILE>)

in grub
cryptkey=rootfs:<CRYPT_FILE> at the end of cmdline (GRUB_CMDLINE_LINUX="cryptdevice=UUID=<UUID>:cryptlvm root=/dev/mapper/vg0-root cryptkey=rootfs:<CRYPT_FILE>"

chmod 600 /boot/initramfs-linux*
mkinitcpio -p linux
grub-mkconfig -o /boot/grub/grub.cfg
chmod 700 /boot

### Hosts
in /etc/hosts
127.0.0.1 localhost
::1 localhost
127.0.1.1 myhostname.localdomain myhostname

### initramfs
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 filesystems fsck dhcpcd wpa_supplicant connman)
mkinitcpio -p linux

### password
passwd

### grub
/etc/default/grub
GRUB_ENABLE_CRYPTODISK=y
e=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:cryptlvm root=/dev/vg/root ..."
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<UUID>:cryptlvm root=<ROOT> rw intel_pstate=no_hwp" # UUID of /dev/sda1 on which LUKS is stationed (blkid)

mkdir /efi
mount (D1) on /efi
grub-install --target=x86_64-efi --efi-directory=/efi
grub-mkconfig -o /boot/grub/grub.cfg
GRUB_TIMEOUT=15
GRUB_GFXMODE=1024x768
grub-mkconfig -o /boot/grub/grub.cfg

## Post install
### ssh
pacman -S openssh-runit
login as root, enable ssh

### Add user
useradd -m <USER>
passwd <USER>
sudo visudo ----> add <USER> ALL=(ALL) ALL

### Network
iwd iwd-runit connmanctl connman-runit
connmanctl config <ETHERNET> --ipv4 manual <> <> <>
connmanctl config <ETHERNET> --nameservers 1.1.1.1 1.0.0.1

### Shell
chsh -s $(which zsh)

### visudo
at the end add
Defaults passwd_timeout=0
Defaults editor=/usr/bin/nvim

Cmnd_Alias UPDATE_CMDS = /usr/bin/pacman, /usr/bin/yay
Cmnd_Alias OTHER_CMDS = /usr/bin/iotop, /usr/bin/wpa_cli, /usr/bin/mount

<USER> ALL=(ALL) NOPASSWD: UPDATE_CMDS, OTHER_CMDS

### pacman
/etc/pacman.conf
Color
CheckSpace
VerbosePkgLists
ILoveCandy
pacman -S archlinux-mirrorlist artix-mirrorlist

### aur helper
cd ; sudo git clone https://aur.archlinux.org/yay-git.git
cd yay-git
makepkg -si
cd ; sudo rm -rf yay-git

### python
yay -S python39
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3.9 get-pip.py
python get-pip.py

### Basic packages
#### editor
yay -S neovim vim
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
:PlugInstall
pip install neovim
pacman -S python2
enable pip on python2
    curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py
    python2 get-pip.py
#### bluetooth
yay -S bluez-runit bluez bluez-utils
rfkill block bluetooth
rfkill unblock bluetooth
#### other
yay -S git zathura zathura-djvu zathura-ps zathura-pdf-mupdf mpd mpv ncmcpp newsboat picom tmux weechat feh htim lf nsxiv firefox unzip vimv wget parcellite-git pass xorg-xsetroot xclip dash dashbin bc fdupes ffmpeg imagemagick hdparm ibus maim playerctl
