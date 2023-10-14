# OpenBSD Installation Guide


## Devices and partitions 

| Device | Description                                                         |
|--------|---------------------------------------------------------------------|
| sdX    | The Xth SCSI-like disk (i.e.: SATA disks connected to [ahci(4)](https://man.openbsd.org/ahci) interfaces,<br> USB drives, disk arrays connected to RAID controllers) |
| wdX    | The Xth IDE-like disk (i.e., IDE, SATA, etc., connected to [wdc(4)](https://man.openbsd.org/wdc) or <br>[pciide(4)](https://man.openbsd.org/pciide) interfaces) |
| rsdX   | The Xth raw SCASI-like disk                                         | 
| rwdX   | The Xth raw IDE-like disk                                           |

| Partition | Description                           |
|-----------|---------------------------------------|
| a         | Boot disk's `a` is the root partition |
| b         | Boot disk's `b` is the swap partition |
| c         | It is always the entire disk          |


## Full disk encryption

### OpenBSD < 7.3

Boot the installation media and drop to the shell by choosing `S`.

Identify the disk that will be used to install the OS on.
```
dmesg | grep sd
```

In this example, we will use `sd0` while `sd1` is the bootable USB drive 
containing the image.

Make the device for `sd0`.
```
cd /dev && sh MAKEDEV sd0
```

Purge the disk with random data.
```
dd if=/dev/urandom of=/dev/rsd0c 
```

Create GPT.
```
fdisk -gy -b 960 sd0
```

Create the partition layout.
```
disklabel -E sd0
	a a
	"enter"
	*
	RAID
	w
	q
```

Use [bioctl(8)](https://man.openbsd.org/bioctl) to create the `CRYPTO` volume 
(in this example `sd2`. This will prompt for a passphrase for unlocking it. Keep 
track of the new volume's number as it is where the system will be installed.
```
bioctl -c C -l sd0a softraid0
```

Make a device for the new `CRYPTO` volume (`sd2` in this example).
```
sh MAKEDEV sd2
```

Zero out the first MB to clear the garbage located where the master boot record 
and disklabel are expected to be.
```
dd if=/dev/zero of=/dev/rsd2c bs=1m count=1
```

Restart the installer using the script and use `sd2` as the root disk.


### OpenBSD >= 7.3 

Since version 7.3, the installer provides interactive support for setting up 
full disk encryption. When the installer asks if the disk should be encrypted 
type the volume's name (e.g., `sd0`) and keep track of the newly generated 
CRYPTO volume (e.g., `sd2`). Later, use this CRYPTO volume (i.e., `sd2`) as the 
root disk.


## Installing the system 

Follow the installer and add a user, disable `sshd` and enable `xenodm`. 

Also,remember to use the new CRYPTO volume (i.e., `sd2` in this example) as the 
root disk. For partitioning use GPT and the auto layout and then edit by adding 
a ~20GB partition for the `ports` and mount it on `/usr/ports`. You might need 
to resize other partitions to generate free space. Also, if the auto layout does 
not utilize the entire volume, resize the partitions to make use of it.

Once the installation is completed, reboot the system and remove the bootable 
USB.


## Post install

### Groups and login classes

Log in as `root` and add your user to the `staff` class and `operator` group.
```
usermod -L staff <user>
usermod -G operator <user>
```

### User privileges

Allow the user to execute with privileges using `doas` by creating 
`/etc/doas.conf` containing:
```
permit persist :wheel
```

### Connect via Wi-Fi

If you don't have access to a wired connection, connect to Wi-Fi.

First identify the wireless network card using `ifconfig` (in this case it is 
`iwm0`) and enable the interface. If the firmware is missing, connect via a 
wired connection and follow the next section first.
```
ifconfig iwm0 up
```

To scan for available networks use: 
```
ifconfig iwm0 scan | more
```

To connect to one of the available networks create `/etc/hostname.iwm0` with the 
following:
```
nwid "<Network>" wpakey "<password>"
dhcp
```

Then enable the connection.
```
sh /etc/netstart iwm0

```

### Essential updates

Log in as <user> and Perform all available updates, using `doas` if needed.
```
fw_update
syspatch
pkg_add -u
```

### Disable annoying stuff

Remove the console from `xenodm`. Edit `/etc/X11/xenodm/Xsetup_0` and comment 
away anything that starts `xconsole`.

Disable the beeps.
```
wsconsctl keyboard.bell.volume=0
echo 'keyboard.bell.volume=0' >> /etc/wsconsctl.conf
```

### Tune *fstab*

Edit `/etc/fstab` to slightly bump up the performance and alter every partition 
except for `swap` by adding the following.
```
noatime,softdep
```

Also add `wxallowed` for `/usr/ports`.

### Tune the *staff* login class

Bump up the resources for the `staff` login class by editing `/etc/login.conf` 
as follows.
```
staff:\
	:datasize-cur=2048:\
	:datasize-max=infinity:\
	:maxproc-cur=512:\
	:maxproc-max=1024:\
	:openfiles-cur=4096:\
	:openfiles-max=8192:\
	:stacksize-cur=32M:\
	:ignorenologin:\
	:requirehome@:\
	:tc=default:
```

### Tune the kernel's parameters

Then bump up the kernel's parameters as following. In this example, we assume a 
system with 16GB of RAM so we utilize 4GB for `shmmax` and 4MB for `shmall`. 
Create `/etc/sysctl.conf` with the following:
``` 
# shared memory limits
kern.shminfo.shmall=4194304 
kern.shminfo.shmmax=4121952256
kern.shminfo.shmmni=4096

# semaphores
kern.shminfo.shmseg=2048
kern.seminfo.semmns=4096
kern.seminfo.semmni=1024

# UDP and ICMP
net.inet.udp.recvspace=262144
net.inet.udp.sendspace=262144
net.inet.icmp.errppslimit=1000

# files and caching
kern.maxproc=32768
kern.maxfiles=65535
kern.bufcachepercent=90
kern.maxvnodes=262144
kern.somaxconn=2048

# enable audio recording
kern.audio.record=1

# enable video recording
kern.video.record=1

# enable hyperthreading
hw.smt=1

# aperture for X11
machdep.allowaperture=2
```

### Power management

If the system is installed on a laptop, enable power management.
```
rcctl enable apmd
rcctl set apmd flags -A 
rcctl start apmd
```

At this point you should have a functional system. Reboot for all changes to 
take effect and keep reading if you need a riced up graphical environment.


## Graphical environment setup and *rice*

### Packages 

The graphical interface is based on the `i3` window manager. To set it up 
install the following:
```
pkg_add picom thunar thunar-archive thunar-media-tags thunar-vcs file-roller 
	tumbler samba gvfs-smb e2fsprogs ntfs_3g avahi i3 i3lock 3status dmenu 
        lxappearance arc-icon-theme arc-theme-solid ffmpeg feh eog thunderbird 
        firefox vim htop wget wheechat dejavusansmono-nerd-fonts noto-emoji 
        noto-cjk liberation-fonts symbola-ttf evince aspell aspell-el colorls 
```

Use the dot files for `i3` `profile` `kshrc` `x11` and `tmux` provided with this 
repo. Each folder contains a readme explaining where each config file should be 
placed.

### Services and daemons

Enable the following services so `thunar` can connect to samba shares.
```
rcctl enable multicast messagebus avahi_daemon
```

### Themes

Using `lxappearance` enable the `Arc-Dark-solid` theme under `Widget` and `Arc` 
under `Icon Theme`.

### Wallpaper

Set your wallpaper with `feh` and `i3` will re-set it each time.
```
feh --bg-fill <path to image>
```
