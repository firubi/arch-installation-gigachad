# Arch installation
## Preparation
The tutorial assumes you are using the systemd hook in mkinitcpio (which recently became the default). Set the timezone: `timedatectl set-timezone Continent/Capital`. On the drive you want to install on, you'll have to wipe it with something like `blkdiscard -f /dev/X`. You can find the drive with `lsblk`. 

## Partitioning and formatting
Use something like `fdisk` to partition. I personally have a boot partition (type 1) of 3G and the rest is a root partition of type Linux x86-64 (type 23). In fdisk, simply type g for a new GPT-table, n for a new partition and t for changing the partition types. You'll have to format the partitions:
```
mkfs.fat -F 32 /dev/efi_system_partition
mkfs.ext4 /dev/root_partition
```

For an encrypted root partition, follow these steps instead:
```
cryptsetup luksFormat /dev/root_partition
cryptsetup open /dev/root_partition root
mkfs.ext4 -L archlinux /dev/mapper/root
mkfs.fat -F32 /dev/efi_system_partition
mkfs.ext4 /dev/root_partition
```

## Mounting 
Using this partition scheme, this is super simple. Make sure to mount root before boot:
```
mount /dev/root_partition /mnt
mkdir /mnt/boot
mount -o dmask=0077,fmask=0177 /dev/efi_system_partition /mnt/boot
```

For an encrypted root partition, follow these steps instead:
```
mount /dev/mapper/root /mnt
mkdir /mnt/boot
mount -o dmask=0077,fmask=0177 /dev/efi_system_partition /mnt/boot
```

You can mount efi to anything really, like /efi or /boot/efi if you prefer. 

## Installing system
We'll keep it simple for now:
```
pacstrap -K /mnt base 
```
We'll install everything else after chrooting! In order to generate fstab file, use `genfstab`: `genfstab -U /mnt >> /mnt/etc/fstab`. Edit it to have these mount options:
```
UUID=0a3407de-014b-458b-b5c1-848e92a327a3 /     ext4 defaults                                           0      1
UUID=CBB6-24F2                            /boot vfat defaults,nodev,nosuid,noexec,fmask=0177,dmask=0077 0      2
```

## Chroot
The system is essentially installed, and now you just have to configure it. This is just a matter of following the simple steps [here](https://wiki.archlinux.org/title/Installation_guide#Chroot). 
### Time
```
ln -sf /usr/share/zoneinfo/Area/Location /etc/localtime
hwclock --systohc
```
### Locale
First edit /etc/locale.gen and uncomment en_US.UTF-8, then run `locale-gen`. You have to install a text editor, like `nano`. Then you want to edit /etc/locale.conf and insert:
```
LANG=en_US.UTF-8
LC_TIME=nb_NO.UTF-8 #optional, if you have other locales and want 24hrs clock
```
Then edit /etc/vconsole.conf and insert:
```
KEYMAP=no # or whatever your keymap
XKBLAYOUT=no
FONT=eurlatgr
```
Then edit /etc/hostname and give your computer a hostname. 

At this point, install all other packages: `sudo linux mkinitcpio linux-headers linux-firmware amd-ucode networkmanager zram-generator firewalld limine efibootmgr plymouth reflector`. If you plan on using the AUR, you will need additionally `base-devel`.

### Additional remarks
For nvidia users: Edit /etc/mkinitcpio.conf and add `nvidia nvidia_modeset nvidia_uvm nvidia_drm` to MODULE. Then install `nvidia-utils nvidia-open-dkms`, and if you plan on using Steam, enable multilib in /etc/pacman.conf and install `lib32-nvidia-utils`. 
For encrypted root users: your mkinitcpio hooks should look something like this: HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck), most importantly include sd-encrypt.
For plymouth users: add the plymouth hook in mkinitcpio - make sure it is before sd-encrypt.

I also like to uncomment/add Color, ILoveCandy and Verbose.

### Sudo, user
Make an account:
```
useradd -m -G wheel NAME
passwd NAME
passwd -l root
```
Locking root is optional. Edit the wheel group (the top one) by uncommenting it with `EDITOR=nano visudo`.

### Bootloader
I now use Limine, as it's very versatile. For installation, follow the [wiki](https://wiki.archlinux.org/title/Limine). Packages you will need are `efibootmgr` and `limine`. First, make the directory `mkdir -p /boot/EFI/arch-limine`. Then copy the Limine bootloader from /usr/ like this: `cp /usr/share/limine/BOOTX64.EFI /boot/EFI/arch-limine/`. For a boot entry in UEFI, use efibootmgr:
```
efibootmgr \
      --create \
      --disk /dev/sdX \
      --part Y \
      --label "Arch Linux" \
      --loader '\EFI\arch-limine\BOOTX64.EFI' \
      --unicode
```
Where disk refers to the disk (not partiiton), and part refers to the /boot partition on the disk. Finally, in /boot, the limine.conf file must be created and edited. A simple config file will look like this:
```
timeout: 5
remember_last_entry: yes

/Arch Linux
    protocol: linux
    path: boot():/vmlinuz-linux
    cmdline: root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rw zswap.enabled=0
    module_path: boot():/initramfs-linux.img
```
For encrypt root users: instead of `root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` add `rd.luks.name=device-UUID=root root=/dev/mapper/root` at the end of cmdline - the device-UUID is of the partition (eg. /dev/nvme1n1p2) and not encrypted volume (eg. /dev/mapper/root).
For plymouth users: add `quiet splash` at the end of cmdline.

If you want a custom spalsh screen, add this to the config as well:
```
wallpaper: boot():/splash.png
wallpaper_style: stretched
backdrop: 000000
```

### ZRAM
Instead of a swap-partition or swap-file, I use ZRAM. Edit /etc/systemd/zram-generator.conf to include:
```
[zram0]
zram-size = 16384 # or adjust to your system
compression-algorithm = lzo-rle # the default for Fedora, otherwise the default is zstd
```

### Finishing
Just to make sure, run `mkinitcpio -P`

### Done
```
exit
umount -R /mnt
reboot
```

## Post-install
For post-install and tweaks I do after install, check out my [other](https://github.com/firubi/post-install/tree/main) post.
