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

