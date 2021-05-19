# DRAKVUF
DRAKVUF® is a virtualization based agentless black-box binary analysis system. DRAKVUF® allows for in-depth execution tracing of arbitrary binaries (including operating systems), all without having to install any special software within the virtual machine used for analysis.

For more details about it, [visit here](https://drakvuf.com/).

## Installation guide

The system has been mainly tested on Ubuntu 20.04 LTS.

Installing required dependencies.
```
sudo apt-get install wget git bcc bin86 gawk bridge-utils iproute2 libcurl4-openssl-dev bzip2 libpci-dev build-essential make gcc clang libc6-dev linux-libc-dev zlib1g-dev libncurses5-dev patch libvncserver-dev libssl-dev libsdl-dev iasl libbz2-dev e2fslibs-dev git-core uuid-dev ocaml libx11-dev bison flex ocaml-findlib xz-utils gettext libyajl-dev libpixman-1-dev libaio-dev libfdt-dev cabextract libglib2.0-dev autoconf automake libtool libjson-c-dev libfuse-dev liblzma-dev autoconf-archive kpartx python3-dev python3-pip golang python-dev libsystemd-dev nasm -y
```
Some python packages are unfortunately quite old in Debian so we will install them with Python's pip3 tool.
```
sudo pip3 install pefile construct
```
Cloning Drakvuf
```
cd ~
git clone https://github.com/tklengyel/drakvuf
cd drakvuf
git submodule update --init
cd xen
./configure --enable-githttp --enable-systemd --enable-ovmf --disable-pvshim
make -j4 dist-xen
make -j4 dist-tools
make -j4 debball
```
To install Xen with dom0 getting 4GB RAM assigned and two dedicated CPU cores (tune it as preferred)

**Note:** Also, you can set RAM size and CPU cores according to you.
```
sudo su
apt-get remove xen* libxen*
dpkg -i dist/xen*.deb
echo "GRUB_CMDLINE_XEN_DEFAULT=\"dom0_mem=4096M,max:4096M dom0_max_vcpus=4 dom0_vcpus_pin=1 force-ept=1 ept=ad=0 hap_1gb=0 hap_2mb=0 altp2m=1 hpet=legacy-replacement smt=0\"" >> /etc/default/grub
echo "/usr/local/lib" > /etc/ld.so.conf.d/xen.conf
ldconfig
echo "none /proc/xen xenfs defaults,nofail 0 0" >> /etc/fstab
echo "xen-evtchn" >> /etc/modules
echo "xen-privcmd" >> /etc/modules
systemctl enable xen-qemu-dom0-disk-backend.service
systemctl enable xen-init-dom0.service
systemctl enable xenconsoled.service
update-grub
```
**Note:**  Also make sure you are running a relatively recent kernel. In Ubuntu 20.04 the 5.10.0-1019-oem kernel has been verified to work, anything newer would also work. Older kernels in your dom0 will not work properly!
```
uname -r
```

Make xen boot before the kernel.
```
cd /etc/grub.d/;mv 20_linux_xen 09_linux_xen
```
Once you are done with these steps, you can finalize your setup.
```
update-grub
reboot
```
Once you are booted into Xen, verify that everything works as such.
```
sudo xen-detect
```
Output should be:
Running in PV context on Xen v4.7

Check Dom 0 is running or not.
```
xl list
```
The output should be something similar:
```
Name                                        ID   Mem VCPUs	State	Time(s)
Domain-0                                     0  4096     2     r-----     614.0
```
## Setup an LVM volume Group

Install lvm2.
```
sudo apt-get install lvm2 -y
```
**Note:** Before creating physical volume (PV), Go inside Disk partition and create a volume. Never give the whole path of disk like /dev/sda otherwise your os will crash. So when you will create a volume its has named like /dev/sd2 or /dev/sd3 and so on. So pick only a free volume then move ahead.

Create physical volume on your free space in my case its /dev/sda2.
```
pvcreate /dev/sda2
```
Create Volume Group
```
vgcreate vg /dev/sda2
```
Create Logical Volume.

**syntex:** Syntax: lvcreate –L<amount of space in GB>G –n<any name for your system> <volume group name previously mentioned>
```
lvcreate –L110G –n windows7-sp1 vg
```

## Install VMM for virtual bridge create

Go to ubuntu software center and install virtual machine manager.




