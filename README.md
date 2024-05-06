# Arch Linux ARM for A10(X) Fusion
Thats guide based on Project Sandcastle repositories and postmarketOS guide. \
**Works with palera1n \ iOS 15.x.** \
Tested on iPhone 7 Plus with iOS 15.8.1 \
I hope to be able to publish an image with the system soon so people don't have to build it manually. Also hopefully there will be more features. You can still contact me on issues or [Telegram](https://t.me/cocoshark)

> [!IMPORTANT]
> The instruction provides that user knows how to work with CLI and knows basic UNIX commands. If not, check out the Arch Linux Wiki, man pages or Wikipedia.
> A native ARM machine (even rooted Android) and x86_64 Linux were used for the build and boot up.

> [!WARNING]
> I am not responsible for your devices and data while working according to these instructions. \
> If you want to try out this experience, be careful with the commands you enter.

## What's working now?
- [x] CPU
- [x] Storage
- [x] Battery
- [x] Power button
- [x] Display (tty)
- [x] USB-Lightning (CDC, USB-Modem)
- [ ] Touchscreen (needs test)
- [ ] Graphical interface (now only VNC)
- [ ] Broadcom BCM4355 (WiFi, Bluetooth)
- [ ] Intel XMM7360 (WWAN)
- [ ] Cameras (no driver)

<details>
<summary>

  ## Create image and boot up 
  </summary>
 
### Download base
On a native device, download [latest multi-platform .tar.gz](https://archlinuxarm.org/about/downloads) from the official ALARM website. Create an image.
```` 
dd if=/dev/zero of=/path/to/ArchLinux.img bs=1M count=<size>
````

Wait for the image to be created and create a file system, for convenience here ext4.
````
mkfs.ext4 -b 2048 -E lazy_itable_init -O ^has_journal /path/to/ArchLinux.img
````

Mount image to folder. Here the root directory with image is `/arch`
````
sudo mount -o loop -t /path/to/ArchLinux.img /arch
```` 
````
sudo losetup /dev/loopX /path/to/ArchLinux.img
sudo mount /dev/loopX /arch
````

`cd` to directory and unpack .tar.gz.
````
cd /arch
tar -xf /path/to/Downloads/ArchLinuxARM-aarch64-latest.tar.gz
````

### Chrooting to ALARM
Mount all needed partitions and enter command.
````
sudo mount -o bind /dev /arch/dev
sudo mount -o bind /dev/pts /arch/dev/pts
sudo mount -t proc proc /arch/proc
sudo mount -t tmpfs none /arch/run
sudo mount -t tmpfs -o mode=0755,size=70% none /arch/tmp
sudo chroot /arch /bin/bash
````

If your output is `command not found` when entering the command, enter
````
export PATH=/bin:/usr/bin:/sbin
````
If internet request is answered by `can't resolv`
````
rm /etc/resolv.conf
echo "nameserver 1.1.1.1" > /etc/resolv.conf
````
And re-entry to chroot

### Preparation system
Update the repositories and system packages
````
pacman -Syyu
````
Install the necessary packages, which can be `neofetch`, `dhcpcd`, `iwd`, `networkmanager`, `i2c-tools`, `usbutils`... etc \
The entire list of packages is available at [Official ALARM Web Site](https://archlinuxarm.org/packages) or search by `pacman -Ss <name>` \
Don't forget to set a password for root or other users. For some packages, you need to start services.
````
systemctl enable NetworkManager
systemctl enable dhcpcd
systemctl enable serial-getty@ttyGS0
````
### hxtouchd
**Drivers for touchscreen.** I'm not sure if they work, not tested, because of Xorg. \
There is a [hx-touch](/hx-touch.tar) archive in the repository, unpack it according to your root system folders. \
`systemctl enable hx-D10` for iPhone 7. \
`systemctl enable hx-D11` for iPhone 7 Plus.

### Write image to device
Unmount the partitions and the image. \
Boot iDevice with jailbreak (palera1n, checkra1n, Taurine, depends on version and device). \
Create a new volume in the APFS container and mount it in a temporary folder.
````
newfs_apfs -A -v ArchLinux -e /dev/disk0s1
mkdir -p /tmp/mnt
mount -t apfs /dev/disk0s1s9 /tmp/mnt
`````

Depending on the version, volume number may vary. If you have a different one, rootfs will not be mounted when using initramfs from the repository. Refer to the instructions for compilation your initramfs below. You can check the volume with command:
````
/System/Library/Filesystems/apfs.fs/apfs.util -p /dev/disk0s1sХ
````
Where `vol=X` is volume number. If output is `ArchLinux`, the corresponding volume is necessary for work. \

Make sure that openssh is installed, you know the root password, know the local IP address of the device and send the image via scp. To change the root password in the terminal, enter -
````
sudo passwd root
scp -O /path/to/ArchLinux.img root@local-IP:/tmp/mnt/
````
Wait for the image recording process to finish.

### Booting
A. Download palera1n[^1] or checkra1n 1337 for macOS \
B. Download palera1n or checkra1n 1337[^2] for Linux \
B. Stop usbmuxd and run with verbose \
``sudo systemctl stop usbmuxd`` (systemd) \
``sudo rc-service usbmuxd stop`` (OpenRC) 
````
sudo usbmuxd -f -p &
````
Download PongoOS from GitHub. Go to directory.
````
git clone https://github.com/checkra1n/PongoOS.git
cd ./PongoOS/scripts
````
Make pongoterm. \
For convenience move to `/opt/local/bin/` <sub>macOS</sub>
or `/sbin/` <sub>Linux</sub>
````
make pongoterm
sudo mv ./pongoterm /opt/local/bin/
````

Enter in terminal (console)
`/path/to/palera1n -P` or `/path/to/checkra1n -P` \
And follow instructions of the program. After successful booting to PongoOS, use pongoterm to send files and further bootin’
````
sudo pongoterm
/send /path/to/dtbpack
fdt
/send /path/to/ramdisk.cpio.gz
ramdisk
/send /path/to/Image.lzma
bootl
````

### Congratulations
You are booting ALARM. \
After a successful boot, you can use tty from a physical keyboard. \
If the above method does not suit you, use the screen (minicom) program on another Linux device.
````
screen /dev/ttyACM0
````

</details>
<details>
<summary>
  
  ## Build own kernel with initramfs 
  </summary>

## Unpack initramfs \ Edit init
On a native device, in the ALARM environment, unpack the initramfs into a convenient folder and edit the init.
````
mkdir ~/ramdisk
cd ~/ramdisk
lsinitcpio -x /boot/iniramfs-linux.img
sudo nano ./init
````

Replace
`"$mount_handler" /new_root` to
````
/bin/mkdir -p /mnt/apfs /mnt/ro /mnt/rw
/bin/mount -t apfs -o ro,relatime,vol=Х /dev/nvme0n1p1 /mnt/apfs
/sbin/losetup /dev/loop0 /mnt/apfs/ArchLinux.img
/bin/mount -t ext4 -o ro /dev/loop0 /mnt/ro
/bin/mount -t tmpfs tmpfs /mnt/rw
/bin/mkdir -p /mnt/rw/data /mnt/rw/work
/bin/mount -t overlay -o lowerdir=/mnt/ro,upperdir=/mnt/rw/data,workdir=/mnt/rw/work overlay /new_root
````
where `vol=X` is number of volume in APFS container

Pack the image and conveniently send it to your computer or device for compilation.
````
sh -c "find . | cpio  --quiet -o -H newc | gzip -9 > ./ramdisk.cpio.gz"
````

### Compilation
If you are using a native device, check for gcc and glibc.
In the case of x86_64 or other architecture, install gcc packages for Aarch64.
Copy the repository with the kernel.
````
sudo pacman -S aarch64-linux-gnu-gcc 
git clone https://github.com/corellium/linux-sandcastle.git
cd linux-sandcastle
````

Set envs. Make a defconfig, edit if necessary. Copy the ramdisk to this directory. \
After make defconfig, add CONFIG_USB_G_SERIAL to .config for support USB CDC.
````
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make hx_h9p_defconfig
cp /path/to/ramdisk.cpio.gz .
````
Start compilation
````
make -j4
````

If you get error like
`multiple definition of 'yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here` \
Then use\roll back gcc and gcc-libs to version 8.3.0, there is no problem. \
Here links on Arch Linux Archive to download old version GCC.
| Links   | GCC                     | GCC-libs            |
| ---     | ---                     | ---                 |
| Aarch64 | [Download](http://tardis.tiny-vps.com/aarm/packages/g/gcc/gcc-8.3.0-1-aarch64.pkg.tar.xz) | [Download](http://tardis.tiny-vps.com/aarm/packages/g/gcc-libs/gcc-libs-8.3.0-1-aarch64.pkg.tar.xz) |
| x86_64  | [Download](https://archive.archlinux.org/packages/g/gcc/gcc-8.3.0-1-x86_64.pkg.tar.xz) | [Download](https://archive.archlinux.org/packages/g/gcc-libs/gcc-libs-8.3.0-1-x86_64.pkg.tar.xz) |

Wait for the compilation to finish

### Final commands
````
./dtbpack.sh
lzma -z --stdout arch/arm64/boot/Image > ./Image.lzma
````

Use the resulting files `dtbpack`, `ramdisk.cpio.gz` and `Image.lzma` for boot. The steps are described in the [Booting](#booting) stage.
</details>

[^1]: palera1n is used for jailbreak iDevices with iOS 15.x, you can download this from [GitHub](https://github.com/palera1n/palera1n/releases)
[^2]: checkra1n same thing like palera1n, but 1337 version is just boot PongoOS, which can be used with iOS 15.x, even 16.x. [Download](https://checkra.in/1337) \
You can use usual checkra1n if you running iOS 14. [WebSite](https://checkra.in/releases)

<details>
  <summary>
  
  ## Additional information</summary>
### Check battery status
````
upower -i /org/freedesktop/UPower/devices/battery_battery
````

### Configure Ethernet with NetworkManager
Check the status of services
````
systemctl status NetworkManager
systemctl status dhcpcd
````

Create a connection and assign IP addresses for PC and iDevice. \
Set the manual mode and “UP” the connection.
````
nmcli con add con-name "USB Static" ifname usb0 type ethernet ip4 172.16.1.1/24 gw4 172.16.1.2
nmcli con mod "USB Static" ipv4.method manual
ncmli con up "USB Static" ifname usb0
````

You can check the status by entering the command \
`nmcli con show` \
After that, set static IP addresses on the PC side, where \
| IP address (PC)  | Netmask       | Gateway (iDevice) |
| ---              | ---           | ---               |
| 172.16.1.2       | 255.255.255.0 | 172.16.1.1        |

![photo where left is CDC, right is SSH](/CDC_SSH.jpg)
</details>

## Thanks to
[Linus Torvalds](https://github.com/torvalds) for Linux \
[Arch Linux](https://github.com/archlinux) for Linux distribution \
[Corellium](https://github.com/corellium) for kernel fork \
[checkra1n](https://github.com/checkra1n) and [palera1n](https://github.com/palera1n) for jailbreak \
[onny](https://project-insanity.org/author/onny/) from project-insanity for postmarketOS guide and motivation
