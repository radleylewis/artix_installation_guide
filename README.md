# Artix Installation Guide

Good morning, good afternoon or good evening, wherever you are reading this from. These installation instructions form the foundation of the Artix system that I use on my own Librebooted T480.

The Artix Linux documentation can be found [here](https://wiki.artixlinux.org/Main/Installation).

This guide uses the following stack:

- [OpenRC](https://wiki.gentoo.org/wiki/OpenRC): A dependency-based init system (Artix is systemd-free).
- [btrfs](https://btrfs.readthedocs.io/en/latest/): A feature-rich, copy-on-write filesystem for Linux.
- [LUKS](https://gitlab.com/cryptsetup/cryptsetup/): Full disk encryption based on the dm-crypt kernel module.
- [zswap](https://www.kernel.org/doc/html/latest/admin-guide/mm/zswap.html): A compressed write-back cache for swap pages, built into the Linux kernel.
- [Qtile](https://qtile.org/): A full-featured, hackable tiling window manager written and configured in Python.
- [ly](https://github.com/fairyglade/ly): A lightweight TUI display manager.

> [!note] **BIOS vs UEFI**
> This guide includes instructions for both BIOS/SeaBIOS and UEFI systems. Steps that differ between the two are clearly marked with **[BIOS]** and **[UEFI]** labels.

> [!note] **btrfs subvolumes**
> This guide creates the following subvolumes:
>
> | Subvolume    | Mount point   | Purpose                                  |
> | ------------ | ------------- | ---------------------------------------- |
> | `@`          | `/`           | root                                     |
> | `@home`      | `/home`       | user files survive a root rollback       |
> | `@snapshots` | `/.snapshots` | required by snapper as its own subvolume |
> | `@log`       | `/var/log`    | logs persist across rollbacks            |
> | `@cache`     | `/var/cache`  | package cache, excluded from snapshots   |

> [!note] **zswap vs zram**
> This guide uses zswap with a swap partition. zswap is a kernel-level compressed cache that sits in front of your swap partition, compressing pages in RAM before they are written to disk. It requires no additional packages. zswap and zram are **not compatible** - do not enable both.

> [!note] **Partition labels and LUKS**
> The official Artix docs use partition labels (ROOT, HOME, SWAP) for unencrypted setups. Since this guide uses LUKS, the encrypted container is opaque from the outside and labels are not meaningful. The boot process relies on the LUKS UUID in the GRUB cmdline instead.

> [!note] **OpenRC service packages**
> On Artix, many packages require a separate `-openrc` config package to include the OpenRC service file. The base package alone will not register a service.

Let's get started!

---

## Step 1: Creating a Bootable Artix Media Device

Here we will follow the Artix documentation:

i. Acquire an installation image [here](https://iso.artixlinux.org/isos.php) - select the **OpenRC** variant;

ii. Verify the signature on the downloaded ISO image;

iii. Write your ISO to a USB (check out [this](https://www.scaler.com/topics/burn-linux-iso-to-usb/) guide); and,

iv. Insert the USB into your target device and boot into it. Log in as `root`.

> [!tip] OPTIONAL: Increase the console font size
>
> ```bash
> setfont latarcyrheb-sun32
> ```

---

## Step 2: Setting Up Our System with the Artix ISO

> [!tip] OPTIONAL: SSH into your target machine
>
> - Set a password for the ISO root user with `passwd`;
> - Start the ssh service with `rc-service openssh start`; and,
> - Obtain your IP address with `ip addr show` (you can now ssh in from another machine).

i. Set the console keyboard layout (US by default):

- list available keymaps with `ls -R /usr/share/kbd/keymaps`; and,
- load your keymap with `loadkeys <your-keymap>`.

ii. Verify your boot mode - the following returns nothing or an error on BIOS, and a list of variables on UEFI:

```bash
ls /sys/firmware/efi/efivars
```

iii. Connect to the internet:

- the Artix ISO ships with `connman` already running (wired connections are automatic); and,

- confirm your connection with `ping -c 2 artixlinux.org`.

> [!tip] OPTIONAL: Connect to WiFi
>
> ```bash
> connmanctl
> enable wifi
> scan wifi
> agent on
> services
> connect wifi_<tab to complete>
> quit
> ```

iv. Update the system clock:

```bash
rc-service ntpd start
```

v. Partition your disk:

- list your disks with `lsblk`; and,
- open your target disk with `cfdisk /dev/nvme0n1`.

> [!warning] The following step will delete all data on the target disk. Ensure you have selected the correct disk before proceeding.

**[BIOS]** Select `dos` label type and create the following partitions:

| Partition        | Size            | Type             |
| ---------------- | --------------- | ---------------- |
| `/dev/nvme0n1p1` | 2MB             | BIOS Boot        |
| `/dev/nvme0n1p2` | Match your RAM  | Linux swap       |
| `/dev/nvme0n1p3` | Remaining space | Linux filesystem |

> [!note] On a BIOS/MBR dos layout, cfdisk has no dedicated BIOS Boot type. Leave the 2MB partition as a primary Linux partition - GRUB will write to it correctly regardless.

**[UEFI]** Select `gpt` label type and create the following partitions:

| Partition        | Size            | Type             |
| ---------------- | --------------- | ---------------- |
| `/dev/nvme0n1p1` | 1GB             | EFI System       |
| `/dev/nvme0n1p2` | Match your RAM  | Linux swap       |
| `/dev/nvme0n1p3` | Remaining space | Linux filesystem |

Then **Write** and **Quit**.

vi. Format your partitions:

```bash
# swap
mkswap -L SWAP /dev/nvme0n1p2
swapon /dev/nvme0n1p2
```

**[UEFI only]** Format the EFI partition:

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

```bash
# set up LUKS encryption
cryptsetup luksFormat /dev/nvme0n1p3
cryptsetup luksOpen /dev/nvme0n1p3 main

# format as btrfs inside the LUKS container
mkfs.btrfs -L MAIN /dev/mapper/main
```

vii. Create btrfs subvolumes:

```bash
mount /dev/mapper/main /mnt
cd /mnt

btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @snapshots
btrfs subvolume create @log
btrfs subvolume create @cache

cd -
umount /mnt
```

viii. Mount subvolumes:

```bash
# root
mount -o noatime,compress=zstd,discard=async,subvol=@ /dev/mapper/main /mnt

# create mount points
mkdir -p /mnt/{home,.snapshots,var/log,var/cache}

# home
mount -o noatime,compress=zstd,discard=async,subvol=@home /dev/mapper/main /mnt/home

# snapshots
mount -o noatime,compress=zstd,discard=async,subvol=@snapshots /dev/mapper/main /mnt/.snapshots

# log
mount -o noatime,compress=zstd,discard=async,subvol=@log /dev/mapper/main /mnt/var/log

# cache
mount -o noatime,compress=zstd,discard=async,subvol=@cache /dev/mapper/main /mnt/var/cache
```

ix. **[UEFI only]** Mount the EFI partition:

```bash
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

x. Install the base system:

```bash
basestrap /mnt base base-devel openrc elogind-openrc linux linux-firmware
```

xi. Generate the filesystem table:

```bash
fstabgen -U /mnt >> /mnt/etc/fstab
```

Verify with `cat /mnt/etc/fstab` to check that all subvolumes, swap, and boot entries are present and correct before continuing (if using UEFI).

xii. Change root into the new system:

```bash
artix-chroot /mnt
```

You are now working within your new Artix base system (i.e. not from the ISO) and you will now see that your prompt starts with `#`. Great work so far!

---

## Step 3: Working Within Our New Base System

We are now working within our Artix system on our device, but it's important to note that we can't yet reboot our machine. Let's continue with a few steps that we need to repeat again (such as setting our root password, timezones, keymaps and language) given the previous settings were in the context of our ISO.

> [!note] You are now running as root inside the chroot. Do not use `sudo` in this section.

i. Configure the timezone:

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```

ii. Configure the hardware clock:

```bash
hwclock --systohc
```

iii. Configure locales - in `nvim /etc/locale.gen` uncomment your locale (e.g. `en_US.UTF-8`), write and exit, then run:

```bash
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "KEYMAP=us" >> /etc/vconsole.conf
```

iv. Configure the hostname:

```bash
# T480 will be my hostname, swap in yours
echo "T480" > /etc/hostname
```

Add the same hostname to `/etc/conf.d/hostname`:

```bash
nvim /etc/conf.d/hostname
hostname="T480"
```

Add matching entries to `/etc/hosts`:

```bash
nvim /etc/hosts
127.0.0.1    localhost
::1          localhost
127.0.1.1    T480.localdomain T480
```

v. Set the root password:

```bash
passwd
```

vi. Create a user (replace `rad` with your preferred username):

```bash
useradd -m -G wheel rad
passwd rad
mkdir /etc/sudoers.d
chmod 755 /etc/sudoers.d   # readable, executable by all; writable by root only
echo "rad ALL=(ALL) ALL" >> /etc/sudoers.d/rad
chmod 0440 /etc/sudoers.d/rad   # read-only for root and group; visudo will refuse otherwise
```

vii. Set the mirrorlist (substitute Singapore with your location):

```bash
pacman -S reflector
reflector -c Singapore -a 12 --sort rate --save /etc/pacman.d/mirrorlist
```

viii. Enable the Arch Linux repositories:

Artix does not mirror every Arch package. Adding the Arch repos gives access to packages like `qtile` that are not yet in the Artix repos.

```bash
pacman -S artix-archlinux-support
```

Add the following to `/etc/pacman.conf` **after** your existing Artix repos:

```
[extra]
Include = /etc/pacman.d/mirrorlist-arch

[multilib]
Include = /etc/pacman.d/mirrorlist-arch
```

Then update:

```bash
pacman -Syu
```

ix. Install core packages:

```bash
pacman -Syu \
  linux-headers \          # kernel build headers
  btrfs-progs \            # btrfs utilities
  cryptsetup \             # LUKS encryption
  networkmanager \         # network management
  networkmanager-openrc \  # OpenRC service file
  network-manager-applet \ # system tray applet
  openssh \                # SSH server/client
  openssh-openrc \         # OpenRC service file
  sudo \                   # privilege escalation
  neovim \                 # text editor
  git \                    # version control
  firewalld \              # firewall frontend
  firewalld-openrc \       # OpenRC service file
  reflector \              # mirror ranking
  acpid \                  # ACPI event daemon
  acpid-openrc \           # OpenRC service file
  cronie \                 # cron scheduler
  cronie-openrc \          # OpenRC service file
  grub \                   # bootloader
  elogind \                # login session manager
  dbus \                   # inter-process messaging
  dbus-openrc \            # OpenRC service file
  polkit \                 # privilege authorisation
  xdg-user-dirs \          # standard home directories
  udevil \                 # USB automounting
  udevil-openrc            # OpenRC service file
```

**[UEFI only]** Also install:

```bash
pacman -S efibootmgr       # EFI boot manager
```

x. Install CPU microcode:

- **Intel:** `pacman -S intel-ucode`
- **AMD:** `pacman -S amd-ucode`

xi. Install audio:

```bash
pacman -S \
  pipewire \             # audio server
  pipewire-audio \       # audio support
  pipewire-alsa \        # ALSA routing
  pipewire-pulse \       # PulseAudio replacement
  pipewire-jack \        # JACK replacement
  wireplumber \          # session manager
  alsa-utils \           # ALSA utilities
  sof-firmware           # sound card firmware
```

xii. Install Bluetooth:

```bash
pacman -S \
  bluez \                # Bluetooth stack
  bluez-utils \          # Bluetooth utilities
  bluez-openrc           # OpenRC service file
```

xiii. Install Qtile, ly, and Wayland dependencies:

```bash
pacman -S \
  ly \                      # display manager
  ly-openrc \               # OpenRC service file
  qtile \                   # window manager
  python-pywlroots \        # Wayland backend
  python-pywayland \        # Wayland bindings
  python-xkbcommon \        # keyboard handling
  xorg-xwayland \           # X11 compatibility
  wlr-randr \               # display management
  swaylock \                # screen locker
  swaybg \                  # wallpaper setter
  lxqt-policykit \          # polkit auth agent
  xdg-desktop-portal \      # portal base layer
  xdg-desktop-portal-wlr \  # screen sharing, file pickers
  python-psutil \           # system stats widgets
  qt5-wayland \             # Qt5 Wayland support
  qt6-wayland \             # Qt6 Wayland support
  grim \                    # screenshot tool
  slurp \                   # region selector
  wl-clipboard \            # Wayland clipboard
  xclip \                   # X11 app clipboard
  mako \                    # notification daemon
  brightnessctl \           # brightness control
  playerctl                 # media key support
```

> [!note] `rofi-wayland` is an AUR package and will be installed via `paru` after the first boot in Step 4.

> [!note] If `qtile` is not found, ensure the Arch repos were added correctly in Step 3 (viii) and run `pacman -Syu` before retrying.

> [!note] To verify the correct path for the polkit agent after installation, run `find /usr -name "lxqt-policykit-agent"` and update your Qtile autostart accordingly.

xiv. Install a file manager and supporting utilities:

```bash
pacman -S \
  thunar \               # file manager
  gvfs \                 # trash, USB mounting
  gvfs-mtp \             # Android device support
  pavucontrol            # audio volume control
```

xv. Install fonts, terminal and browser:

```bash
pacman -S \
  man-db \               # manual pages
  man-pages \            # manual content
  texinfo \              # info pages
  ttf-firacode-nerd \    # nerd font
  kitty \                # terminal emulator
  firefox                # web browser
```

xvi. Configure USB automounting:

```bash
rc-update add devmon default
```

> [!note] `udevil` provides the `devmon` service which watches for USB devices and mounts them automatically under `/media`. Thunar will show mounted devices in its sidebar without any further configuration.

xvii. Configure the firewall:

```bash
rc-update add firewalld default
rc-service firewalld start
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

xviii. Configure mkinitcpio for LUKS + btrfs:

Edit `/etc/mkinitcpio.conf` and set your MODULES and HOOKS lines to:

```
MODULES=(btrfs)
HOOKS=(base udev autodetect modconf kms keyboard keymap block encrypt filesystems)
```

> [!note] `fsck` is removed from HOOKS (it is not required for btrfs). `consolefont` is removed as it requires `terminus-font` and serves no purpose at the LUKS prompt. This guide uses a `udev`-based initramfs. If you are using a `systemd`-based initramfs, replace `encrypt` with `sd-encrypt`.

> [!tip] OPTIONAL: If you use an external USB keyboard and need it available at the LUKS prompt, add `usbhid` and `atkbd` to MODULES:
>
> ```
> MODULES=(btrfs usbhid atkbd)
> ```

Regenerate the initramfs:

```bash
mkinitcpio -P
```

xix. Configure GRUB:

**[BIOS]** Install GRUB to your disk:

```bash
grub-install --recheck /dev/nvme0n1
```

**[UEFI]** Install GRUB to the EFI partition:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

Get the UUID of your LUKS partition and append it to the grub defaults file for reference:

```bash
blkid -s UUID -o value /dev/nvme0n1p3 >> /etc/default/grub
```

Edit `/etc/default/grub` and set the following - replacing `<uuid>` with the value appended at the bottom of the file, then delete that line:

```
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet zswap.enabled=1 cryptdevice=UUID=<uuid>:main root=/dev/mapper/main"
GRUB_GFXMODE=1920x1080x32
GRUB_GFXPAYLOAD_LINUX=keep
```

> [!note] `GRUB_ENABLE_CRYPTODISK=y` is required because `/boot` resides inside the LUKS-encrypted partition. Without it, GRUB cannot read the kernel and initramfs at boot. This means you will be prompted for your LUKS password **twice** on boot (once by GRUB to load the kernel, and once by the initramfs to mount root).

> [!note] `zswap.enabled=1` makes the zswap kernel feature explicit. zswap sits in front of your swap partition, compressing pages in RAM before writing them to disk.

Generate the GRUB config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

xx. Set up standard home directories:

```bash
xdg-user-dirs-update
```

xxi. Enable services:

```bash
rc-update add NetworkManager default
rc-update add bluetoothd default
rc-update add openssh default
rc-update add ly default
rc-update add firewalld default
rc-update add acpid default
rc-update add cronie default
rc-update add dbus default
rc-update add devmon default
```

xxii. Set up cron jobs for periodic maintenance:

```bash
# weekly SSD trim
echo "@weekly root fstrim -av" >> /etc/cron.d/fstrim

# weekly mirrorlist update - substitute Singapore with your location
echo "@weekly root reflector -c Singapore -a 12 --sort rate --save /etc/pacman.d/mirrorlist" >> /etc/cron.d/reflector
```

xxiii. Reboot:

```bash
exit
umount -R /mnt
reboot
```

> [!note] **BIOS/SeaBIOS users:** After installation, press Escape at POST to access the SeaBIOS boot menu and select your NVMe. A brief blank screen before the GRUB menu is normal - it is the handoff from SeaBIOS to GRUB.

---

## Step 4: Tweaking Our New Artix System

Upon successful decryption, ly will start. Log in with the user you created earlier and select the Qtile Wayland session.

i. Connect to WiFi if needed:

```bash
nmtui
```

ii. Install [paru](https://github.com/Morganamilo/paru):

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

iii. Install rofi-wayland:

```bash
paru -S rofi-wayland
```

iv. Configure Qtile:

Qtile's config lives at `~/.config/qtile/config.py`. Create the directory if it does not exist:

```bash
mkdir -p ~/.config/qtile
```

On first login Qtile will use its built-in default config if none is present, which is functional but minimal. I will cover Qtile configuration in a separate video.

---

## Recovery

If you cannot boot into your system, boot from your Artix USB, open the encrypted partition and chroot back in:

```bash
cryptsetup luksOpen /dev/nvme0n1p3 main
mount -o noatime,compress=zstd,discard=async,subvol=@ /dev/mapper/main /mnt
mount -o noatime,compress=zstd,discard=async,subvol=@home /dev/mapper/main /mnt/home
artix-chroot /mnt
```

From here you can fix your GRUB config, reinstall packages, or regenerate the initramfs as needed.

---

## Optional Further Steps

i. Install `power-profiles-daemon`:

```bash
sudo pacman -S power-profiles-daemon
sudo rc-update add power-profiles-daemon default
sudo rc-service power-profiles-daemon start
```

ii. Install [auto-cpufreq](https://github.com/AdnanHodzic/auto-cpufreq):

```bash
paru -S auto-cpufreq
sudo rc-update add auto-cpufreq default
sudo rc-service auto-cpufreq start
```

iii. Install `brave-browser`:

```bash
paru -S brave-browser
```

iv. Add snapper for btrfs snapshots (when ready):

```bash
sudo pacman -S snapper
sudo snapper -c root create-config /
```

> [!note] Snapper expects to manage `/.snapshots` itself. If it complains that `/.snapshots` already exists, run the following to let snapper create and own the subvolume itself:
>
> ```bash
> umount /.snapshots
> btrfs subvolume delete /.snapshots
> snapper -c root create-config /
> ```

Then install the GRUB integration and pacman hook:

```bash
paru -S grub-btrfs snap-pac grub-btrfs-openrc
rc-update add grub-btrfsd default
rc-service grub-btrfsd start
grub-mkconfig -o /boot/grub/grub.cfg
```

> [!note] `snap-pac` creates a snapshot automatically before and after every pacman operation (i.e. installs, upgrades and removals). `grub-btrfsd` watches for new snapshots and updates the GRUB menu dynamically, so your snapshots will always be available as a boot option without any manual steps. Refer to the [Arch Wiki snapper page](https://wiki.archlinux.org/title/Snapper) for full configuration.

---

That's all folks!
