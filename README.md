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
**Note:** Before creating physical volume (PV), Go inside Disk partition and create a volume. Never give the whole path of disk like /dev/sda otherwise your os will be crashed down. So when you will create a volume then it has named like /dev/sd2 or /dev/sd3 and so on. So pick only a free volume then move ahead.

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
lvcreate -L110G -n windows7-sp1 vg
```

## Install VMM for virtual bridge create

Go to ubuntu software center and install virtual machine manager.
  
## Setup Linux Bridge for guest networking
  
Next we need to set up our system so that we can attach virtual machines to the external network. This is done by creating a virtual switch within dom0. The switch will take packets from the virtual machines and forward them on to the physical network so they can see the internet and other machines on your network.

The piece of software we use to do this is called the Linux bridge and its core components reside inside the Linux kernel. In this case, the bridge acts as our virtual switch. The Debian kernel is compiled with the Linux bridging module so all we need to do is install the control utilities:
  
```
apt-get install bridge-utils
```
  
  Management of the bridge is usually done using the brctl command. The initial setup for our Xen bridge, though, is a "set it once and forget it" kind of thing, so we are instead going to configure our bridge through Debian’s networking infrastructure. It can be configured via /etc/network/interfaces.

Open this file with the editor of your choice. If you selected a minimal installation, the nano text editor should already be installed. Open the file:
  ```
 nano /etc/network/interfaces
  ```
  
  (If you get nano: command not found, install it with apt-get install nano.)

Depending on your hardware you probably see a file pretty similar to this:
  
  ```
    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet dhcp
  ```
  This file is very simple. Each stanza represents a single interface.

Breaking it down, “auto eth0” means that eth0 will be configured when ifup -a is run (which happens at boot time). This means that the interface will automatically be started/stopped for you. ("eth0 is its traditional name - you'll probably see something more current like "ens1", "en0sp2" or even "enx78e7d1ea46da")

“iface eth0” then describes the interface itself. In this case, it specifies that it should be configured by DHCP - we are going to assume that you have DHCP running on your network for this guide. If you are using static addressing you probably know how to set that up.

We are going to edit this file so it resembles such:
  ```
  auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet manual
    
    auto xenbr0
    iface xenbr0 inet dhcp
         bridge_ports eth0
  ```
  As well as adding the bridge stanza, be sure to change dhcp to manual in the iface eth0 inet manual line, so that IP (Layer 3) is assigned to the bridge, not the interface. The interface will provide the physical and data-link layers (Layers 1 & 2) only.

Now restart networking (for a remote machine, make sure you have a backup way to access the host if this fails):
  ```
  service network-manager restart
  ```
  and check to make sure that it worked:
  ```
      brctl show
  ```
  If all is well, the bridge will be listed and your interface will appear in the interfaces column:
  ```
  bridge name     bridge id               STP enabled     interfaces
    xenbr0          8000.4ccc6ad1847d       no              enp2s0
  ```
  Bridged networking will now start automatically every boot.

If the bridge isn't operating correctly, go back and check the edits to the interfaces file very carefully.

  
  ## Create configuration file from which we will create window 7 machine
   Make a config file inside ``/etc/xen/``
 ```
  gedit /etc/xen/win7.cfg
  ```
 Install Windows 7 from your ISO using the following template (tune it as needed):
  ```
  arch = 'x86_64'
name = "windows7-sp1"
maxmem = 1500
memory = 1500
vcpus = 1
maxcpus = 1
builder = "hvm"
boot = "cd"
hap = 1
acpi = 1
on_poweroff = "destroy"
on_reboot = "restart"
on_crash = "destroy"
vnc=1
vnclisten="0.0.0.0"
usb = 1
usbdevice = "tablet"
altp2mhvm = 1
vif = ['type=ioemu,model=e1000,bridge=<your virtual bridge name like in my case it is virbr0>,mac=00:06:5B:BA:7C:01']
disk = [
    'phy:/dev/vg/windows7-sp1,xvda,w', 
    'file:<path to your windows 7 image in my case it is "/home/xen/Desktop/windows7.iso">,hdc:cdrom,r'
]

boot="dc"
sdl=0
vncconsole=1
vncpasswd=''
serial='pty'
  ```
  Now run the below command to create windows7 VM
  
  ```
  xl create /etc/xen/win7.cfg
  ```
  
  Enter LibVmi folder in the drakvuf folder and build it.
  ```
  cd ~/drakvuf/libvmi
./autogen.sh
./configure --disable-kvm
  ```
  Output should look like this:
  ```
  Feature         | Option
----------------|---------------------------
Xen Support     | --enable-xen=yes
KVM Support     | --enable-kvm=no
File Support    | --enable-file=yes
Shm-snapshot    | --enable-shm-snapshot=no
Rekall profiles | --enable-rekall-profiles=yes
----------------|---------------------------

OS              | Option
----------------|---------------------------
Windows         | --enable-windows=yes
Linux           | --enable-linux=yes


Tools           | Option                    | Reason
----------------|---------------------------|----------------------------
Examples        | --enable-examples=yes
VMIFS           | --enable-vmifs=yes        | yes
```
  Build and install LibVMI
  ```
  make
sudo make install
sudo echo "export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:/usr/local/lib" >> ~/.bashrc
```
  Build and install Rekall
  ```
  cd ~/drakvuf/rekall/rekall-core
sudo pip install setuptools
python setup.py build
sudo python setup.py install
  ```
  Now we will create the JSON configuration file for the Windows domain. First, we need to get the debug information for the Windows kernel via the LibVMI vmi-win-guid tool. For example, in the following my domain is named ``windows7-sp1``.
  ```
  $ sudo xl list
Name                                        ID   Mem VCPUs	State	Time(s)
Domain-0                                     0  4024     4     r-----     848.8
windows7-sp1-x86                             7  3000     1     -b----      94.7
$ sudo vmi-win-guid name windows7-sp1-x86
Windows Kernel found @ 0x2604000
	Version: 32-bit Windows 7
	PE GUID: 4ce78a09412000
	PDB GUID: 684da42a30cc450f81c535b4d18944b12
	Kernel filename: ntkrpamp.pdb
	Multi-processor with PAE (version 5.0 and higher)
	Signature: 17744.
	Machine: 332.
	# of sections: 22.
	# of symbols: 0.
	Timestamp: 1290242569.
	Characteristics: 290.
	Optional header size: 224.
	Optional header type: 0x10b
	Section 1: .text
	Section 2: _PAGELK
	Section 3: POOLMI
	Section 4: POOLCODE
	Section 5: .data
	Section 6: ALMOSTRO
	Section 7: SPINLOCK
	Section 8: PAGE
	Section 9: PAGELK
	Section 10: PAGEKD
	Section 11: PAGEVRFY
	Section 12: PAGEHDLS
	Section 13: PAGEBGFX
	Section 14: PAGEVRFB
	Section 15: .edata
	Section 16: PAGEDATA
	Section 17: PAGEKDD
	Section 18: PAGEVRFC
	Section 19: PAGEVRFD
	Section 20: INIT
	Section 21: .rsrc
	Section 22: .reloc
  ```
  The important fields are:
  ```
  PDB GUID: 684da42a30cc450f81c535b4d18944b12
Kernel filename: ntkrpamp.pdb
  ```
  
Generate Rekall profile.
	
```
cd /tmp
rekall fetch_pdb ntkrpamp 684da42a30cc450f81c535b4d18944b12
rekall parse_pdb ntkrpamp > windows7-sp1.rekall.json
sudo mv windows7-sp1.rekall.json /root
```
	
	
Create LibVMI config with Rekall profile:
```
sudo su
printf "windows7-sp1 { \n\
    ostype = \"Windows\"; \n\
    rekall_profile = \"/root/windows7-sp1.rekall.json\"; \n\
}" >> /etc/libvmi.conf
exit
```
	
Test if LibVMI is working or not:
	
```
sudo vmi-process-list windows7-sp1
```
	
Output should be similar to:
	
```
Process listing for VM windows7-sp1-x86 (id=7)
[    4] System (struct addr:84aba980)
[  220] smss.exe (struct addr:85a44020)
[  300] csrss.exe (struct addr:85f67a68)
[  336] wininit.exe (struct addr:8601e030)
[  348] csrss.exe (struct addr:84ba4030)
[  384] winlogon.exe (struct addr:85966d40)
[  444] services.exe (struct addr:8614c030)
[  460] lsass.exe (struct addr:86171030)
[  468] lsm.exe (struct addr:8617b4f8)
[  564] svchost.exe (struct addr:861d9bc8)
[  628] svchost.exe (struct addr:863fb8a8)
[  816] sppsvc.exe (struct addr:86426838)
[  856] svchost.exe (struct addr:854abd40)
[  880] svchost.exe (struct addr:854c5030)
[  916] svchost.exe (struct addr:854d7a70)
[ 1240] svchost.exe (struct addr:8614cb80)
[ 1280] svchost.exe (struct addr:854f7d40)
[ 1608] spoolsv.exe (struct addr:85578660)
[ 1636] svchost.exe (struct addr:85554af0)
[  792] SearchIndexer. (struct addr:8562ac08)
[ 1128] taskhost.exe (struct addr:858d9d40)
[ 1524] dwm.exe (struct addr:857f3a60)
[ 1728] explorer.exe (struct addr:858d9180)
[ 1720] regsvr32.exe (struct addr:8605f398)
[  248] svchost.exe (struct addr:863ed030)
[ 1024] svchost.exe (struct addr:86420390)
[  256] WmiPrvSE.exe (struct addr:854014a0)
```



