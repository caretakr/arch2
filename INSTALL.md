## Installation

1.  Boot a live USB with `latest` release, choose `Arch Linux install medium (x86_64, UEFI)` on screen and wait for live system;

2.  Set keymap (you can use `localectl list-keymaps` to list available keymaps):

        $ loadkeys br-abnt2

3.  Test connectivity with internet:

        $ ping google.com

4.  Open `/dev/xyz` for partitioning (you can check which one using `lsblk`):

        $ gdisk /dev/xyz

        GPT fdisk (gdisk) version 1.0.8

        Partition table scan:
          MBR: not present
          BSD: not present
          APM: not present
          GPT: not present

        Creating new GPT entries in memory.

        Command (? for help): _

5.  Type `n` to create EFI partition with `+256M` on `Last sector` and `ef00` as type:

        Partition number (1-128, default 1):
        First sector (34-167772126, default = 2048) or {+-}size{KMGTP}:
        Last sector (2048-167772126, default = 167772126) or {+-}size{KMGTP}: +256M
        Current type is 8300 (Linux filesystem)
        Hex code or GUID (L to show codes, Enter = 8300): ef00
        Changed type of partition to 'EFI system partition'

        Command (? for help): _

6.  Type `n` to create boot partition with `+768M` on `Last sector`:

        Partition number (2-128, default 2):
        First sector (34-167772126, default = 526336) or {+-}size{KMGTP}:
        Last sector (526336-167772126, default = 167772126) or {+-}size{KMGTP}: +768M
        Current type is 8300 (Linux filesystem)
        Hex code or GUID (L to show codes, Enter = 8300):
        Changed type of partition to 'Linux filesystem'

        Command (? for help): _

7.  Type `n` to create root partition:

        Partition number (3-128, default 3):
        First sector (34-167772126, default = 2099200) or {+-}size{KMGTP}:
        Last sector (2099200-167772126, default = 167772126) or {+-}size{KMGTP}:
        Current type is 8300 (Linux filesystem)
        Hex code or GUID (L to show codes, Enter = 8300):
        Changed type of partition to 'Linux filesystem'

        Command (? for help): _

8.  Type `w` to write partitions and confirm it:

        Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
        PARTITIONS!!

        Do you want to proceed? (Y/N): Y

9.  Format EFI and boot partitions:

        $ mkfs.fat -F 32 /dev/xyz1

        mkfs.fat 4.2 (2021-01-31)

        $ mkfs.brtfs /dev/xyz2

        btrfs-progs v5.15.1
        See http://btrfs.wiki.kernel.org for more information.

        Performing full device TRIM /dev/vda2 (768.00MiB) ...
        NOTE: several default settings have changed in version 5.15, please make sure
        this does not affect your deployments:
        - DUP for metadata (-m dup)
        - enabled no-holes (-O no-holes)
        - enabled free-space-tree (-R free-space-tree)

        Label:              (null)
        UUID:               edfc5a12-0bd2-4ff9-a93c-f68bdab7e83b
        Node size:          16384
        Sector size:        4096
        Filesystem size:    768.00MiB
        Block group profiles:
        Data:             single            8.00MiB
        Metadata:         DUP              38.38MiB
        System:           DUP               8.00MiB
        SSD detected:       no
        Zoned device:       no
        Incompat features:  extref, skinny-metadata, no-holes
        Runtime features:   free-space-tree
        Checksum:           crc32c
        Number of devices:  1
        Devices:
        ID        SIZE  PATH
        1   768.00MiB  /dev/xyz2

10. Create encrypted device for root partition:

        $ cryptsetup luksFormat /dev/xzy3

        WARNING!
        ========
        This will overwrite data on /dev/vda3 irrevocably.

        Are you sure? (Type 'yes' in capital letters): YES

        Enter passphrase for /dev/vda3:
        Verify passphrase:

        cryptsetup luksFormat /dev/vda3  8.58s user 0.78s system 43% cpu 21.377 total

11. Open encrypted device for root partition:

        $ cryptsetup open /dev/xyz3 xyz3

        Enter passphrase for /dev/vda3:

12. Format root partitions:

        $ mkfs.btrfs /dev/mapper/xyz3

        btrfs-progs v5.15.1
        See http://btrfs.wiki.kernel.org for more information.

        Performing full device TRIM /dev/vda3 (79.00GiB) ...
        NOTE: several default settings have changed in version 5.15, please make sure
            this does not affect your deployments:
            - DUP for metadata (-m dup)
            - enabled no-holes (-O no-holes)
            - enabled free-space-tree (-R free-space-tree)

        Label:              (null)
        UUID:               6332837e-4dfc-4e7b-83ad-c94adacfe6f3
        Node size:          16384
        Sector size:        4096
        Filesystem size:    79.00GiB
        Block group profiles:
        Data:             single            8.00MiB
        Metadata:         DUP               1.00GiB
        System:           DUP               8.00MiB
        SSD detected:       no
        Zoned device:       no
        Incompat features:  extref, skinny-metadata, no-holes
        Runtime features:   free-space-tree
        Checksum:           crc32c
        Number of devices:  1
        Devices:
        ID        SIZE  PATH
            1    79.00GiB  /dev/xyz3

13. Create BTRFS subvolumes on a temporary `mount`:

        $ mount /dev/mapper/xyz3 /mnt

        $ btrfs subvolume create /mnt/root

        Create subvolume '/mnt/root'

        $ btrfs subvolume create /mnt/home

        Create subvolume '/mnt/home'

        $ btrfs subvolume create /mnt/var

        Create subvolume '/mnt/var'

        $ umount /mnt

14. Mount all partitions and BTRFS subvolumes:

        $ mount -o noatime,compress=zstd,subvol=root /dev/mapper/xyz3 /mnt
        $ mkdir /mnt/{boot,home,var}
        $ mount /dev/xyz2 /mnt/boot
        $ mkdir /mnt/boot/efi
        $ mount -o shortname=winnt,umask=0077 /dev/xyz1 /mnt/boot/efi
        $ mount -o noatime,compress=zstd,subvol=home /dev/mapper/xyz3 /mnt/home
        $ mount -o noatime,compress=zstd,subvol=var /dev/mapper/xyz3 /mnt/var

15. Install packages from scratch on system:

        $ pacstrap /mnt base

16. Generate `fstab`:

        $ genfstab -U /mnt >> /mnt/etc/fstab

17. `chroot` to Arch:

        $ arch-chroot /mnt

18. Set timezone (you can use `timedatectl list-timezones` to list available keymaps) and syncronize hardware clock:

        $ ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

        $ hwclock --systohc

19. Set locale to English:

        $ sed -i '/^#en_US.UTF-8 UTF-8/s/^#//g' /etc/locale.gen

        $ locale-gen

        Generating locales...
        en_US.UTF-8... done
        Generation complete.

        $ echo "LANG=en_US.UTF-8" >> /etc/locale.conf

20. Set console:

        $ echo "KEYMAP=br" >> /etc/vconsole.conf

21. Set hostname:

        $ echo "arch" >> /etc/hostname

22. Set hosts:

        $ echo "127.0.0.1   localhost arch arch.localdomain" >> /etc/hosts

        $ echo "::1         localhost arch arch.localdomain" >> /etc/hosts

23. Set password for `root`:

        $ passwd

24. Install packages:

        $ pacman -S alsa-firmware bluez btrfs-progs efibootmgr grub intel-ucode linux linux-firmware networkmanager pulseaudio sudo vim

25. Open `/etc/default/grub` and:

    1.  Set `GRUB_CMDLINE_LINUX` (replace `00000000-0000-0000-0000-000000000000` with UUID of `/dev/xyz3` (you can use `blkid -s UUID -o value /dev/xyz3` to get it)) :

            cryptdevice=UUID=00000000-0000-0000-0000-000000000000:xyz3

26. Install GRUB:

        $ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB

        Installing for x86_64-efi platform.
        Installation finished. No error reported.

        $ grub-mkconfig -o /boot/grub/grub.cfg

        Generating grub configuration file ...
        Found linux image: /boot/vmlinuz-linux
        Found initrd image: /boot/intel-ucode.img /boot/initramfs-linux.img
        Found fallback initrd image(s) in /boot:  intel-ucode.img initramfs-linux-fallback.img
        Warning: os-prober will not be executed to detect other bootable partitions.
        Systems on them will not be added to the GRUB boot configuration.
        Check GRUB_DISABLE_OS_PROBER documentation entry.
        Adding boot menu entry for UEFI Firmware Settings ...
        done

27. Enable services:

        $ systemctl enable NetworkManager

        Created symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service → /usr/lib/systemd/system/NetworkManager.service.
        Created symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service → /usr/lib/systemd/system/NetworkManager-dispatcher.service.
        Created symlink /etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service → /usr/lib/systemd/system/NetworkManager-wait-online.service.

        $ systemctl enable bluetooth

        Created symlink /etc/systemd/system/dbus-org.bluez.service → /usr/lib/systemd/system/bluetooth.service.
        Created symlink /etc/systemd/system/bluetooth.target.wants/bluetooth.service → /usr/lib/systemd/system/bluetooth.service.

        $ systemctl enable fstrim.timer

        Created symlink /etc/systemd/system/timers.target.wants/fstrim.timer → /usr/lib/systemd/system/fstrim.timer.

28. Add non-root user and set a new password:

        $ useradd -m caio

        $ passwd caio

        New password: _
        Retype new password: _
        passwd: password updated successfully

29. Add yourself to `sudo` (add `caio ALL=(ALL) ALL` below `root` privilege specification):

        $ visudo

30. Add `btrfs` to `MODULES()` section on `/etc/mkinitcpio.conf` and rebuild ramdisk:

        $ vim /etc/mkinitcpio.conf

        $ mkinitcpio -p linux
