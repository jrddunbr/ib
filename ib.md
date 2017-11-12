## Installation Instructions for setting up InfiniBand on Fedora 26

Please note. While these instructions are specific in parts, they will not help with debugging issues you may encounter. It may seem beginner ready, but this does require a small bit of Linux knowledge to get going. I do not provide much in the way of troubleshooting help.

### Step 1: Install Fedora

#### Download the Iso

I fetched Fedora from [http://mirror.clarkson.edu](http://mirror.clarkson.edu) and went to the Distributions page, and selected [Fedora Iso's](http://mirror.clarkson.edu/fedora/linux/releases/)

Once there, I clicked on ```26```, and then ```Everything``` and then ```x86_64``` and then ```iso```. ([Link to folder](http://mirror.clarkson.edu/fedora/linux/releases/26/Everything/x86_64/iso))

I downloaded the iso file onto my computer (which is running Linux) and then ran ```sync``` to make sure that the file was actually on my computer, and not just in RAM. (This is not strictly necessary).

#### DD the iso to a flash drive

I then copied the installation media onto my flash drive using the ```dd``` program.

This is a very specific instruction, you *cannot* just copy the iso file to a flash drive in Windows. You must use ```dd``` or a similar program to extract the iso onto the disk properly.

To do this, I ran the following commands:

```lsblk``` - This program will tell you the partition layout and block devices available to the system. Generally, look for the device you want by its size (ex, ```/dev/sdb```) and then if necessary, unplug and replug to make sure you have the correct device.

I then use ```dd``` to write the file to the drive. The command looks as below:

```dd if=fedora.iso of=/dev/sdb bs=4M```

```dd``` is the command, ```if=<path to iso>``` is the path to the iso file on your local system, and ```of=/dev/sdb``` is the path to the file. ```bs=4M``` changes the size to copy per iteration, and generally the file will copy fastest at around 4M, but sometimes lower. Depends on your hardware. Bigger (or smaller) does not necessarily regulate how fast it is.

**WARNING** - ```dd``` can directly overwrite your operating system! Be sure you are doing it to the right filesystem, or you could wipe out your running operating system, or other documents. This is completely irrecoverable, even with special software!

Once the ```dd``` program has finished, you want to run ```sync``` again. This will force all of the changes to be applied to the flash drive. After you run this, it is safe to remove the flash drive.

#### Booting the system

**NOTE** - This was hardware specific. You may have to do something different.

On the servers in the lab, you press <kbd>F2</kbd> for Setup, <kbd>F6</kbd> for Boot Menu, and <kbd>F12</kbd> for PXE network boot.

**NOTE** - End hardware specific zone

You may need to enter the BIOS to change the boot order. Once you do that, you need to boot from your bootable flash drive on the system.

#### Installer

From here on, I will call the primary disk ```/dev/sda```

Once the bootable drive has booted, you will see the installer. Just flow through the instructions. I generally start by configuring the disk partitions. I always use the advanded partitioner in basic partition mode (not LVM or BTRFS, etc), which is good for drives up to 2TB. Beyond that, you may need to use UEFI, which requires a special 100MiB partition at the beginning of the disk, which has the /boot folder in it.

Here's a general layout:

#### For MBR based installs:

```
/dev/sda
|- /dev/sda1 - ext4 primary partition with >8GB of space, mounted as /
\- /dev/sda2 - swap primary partition with ~8GB of space. (Optional)
```

#### For UEFI based installs:

```
/dev/sda
|- /dev/sda1 - FAT16/FAT32 primary partition with 100MiB of space, mounted as /boot
|- /dev/sda2 - ext4 primary partition with >8GB of space, mounted as /
\- /dev/sda3 - swap primary partition with ~8GB of space. (Optional)
```

Depending upon whether you have enough RAM for the task at hand, you may want to have swap available. Don't worry, you can also put swap on the ext4 filesystem as well (with a neglible performance hit).

After I have set up the partitions, it's a good idea to set the Mirror.

A mirror is the location that it downloads the files from. You can leave it as the default, but if you want to have the best response, you want to select the closest geographic mirror, considering bandwidth.

At Clarkson University, we have a student-operated [mirror](http://mirror.clarkson.edu) on campus, which has a very fast connection (10G backbone on campus).

The Mirror URL is as follows to use in the dialog that appears when you select the Mirror for Fedora 26:

```
http://mirror.clarkson.edu/fedora/linux/releases/26/Everything/x86_64/os/
```

Once you enter that, click Done. It will take a moment, as it gathers all of the package information.

Next, select the software configuration menu. I selected Server on the left side, and then I selected C/C++ libraries, network utilities, RPM build tools, and other useful tools of which I cannot recall. (Note to self - update this)

Once you do that, go back to the main menu (click done) click Begin Install, and then the magic begins.

Once you do that, it will prompt you for the root password, and a username and password. You will want to make the user administrator. (This means they are added to the wheel group and can use ```sudo``` which is super important later, but so long as you rembmer the root password, you can get anywhere)

After that, you wait for about 30+ minutes as it downloads and installs the packages. The time it takes depends on your disk and network speed for the most part, but a good CPU helps a lot as well.

### First Boot configuration

First comes first, we need to configure the system.

If you haven't already gathered an IP, you will need to get one. Usually, DHCP works out of the box if your network hardware is supported.

Once you know you have a connection, continue.

Run all of the below with the root user (or use ```sudo -i``` on the user account)

```bash
# update the system (this is mandatory or you will hate yourself later)
dnf update

# say yes to all of the prompts (press y and enter)

# install Vim (a text editor), htop (a process monitor), and iperf (a network bandwidth tester)
dnf install -y vim htop iperf

# I tend to disable NetworkManager. It gets in the way too much.
systemctl disable NetworkManager
systemctl stop NetworkManager

# Set the hostname. Delete whatever is there, and then add a hostname which does not start with a number and contains only alphanumeric characters or dashes. No whitespace
vim /etc/hostname

# Reboot to allow changes to occur to the kernel and other hardware that was just updated above
reboot
```

On reboot:

```bash
# When you reboot, there will not be an IP address. Get one via DHCP
# First, we need to determine which ethernet adapter is connected to the internet.
ip a

# Look at the output of this command. lo is always connected, but generally en(p)#(s)#(f)# will be ethernet (where # is generally a number)
# Generally, the first one below lo is what you want. On my system, it's ens2f0
# It will say something like "up" or "lower up" often and will not say "no carrier"

# If it says down, you may just need to poke it with "ip l set ens2f0 up", if it's still down, that is not the correct interface, and it's not connected

# run dhclient on the connected interface to get an IP address
dhclient ens2f0

# where ens2f0 is replaced by the interface that is connected.

# check to see if we have an IP address
ip a l dev ens2f0

# Look for an inet line. The thing directly after inet will be your IP address.

# ex: inet 10.0.3.4/24 ...

# If you have an IP address, you're good to go!
```

### Installing the IB drivers

In general, our hopes are to use OEFD for drivers and such.

We still need to nab some dependencies though.

#### Dependencies

```bash
# Install dependencies for IB
dnf install -y opensm rdma opensm-libs libibumad libibverbs libibverbs-devel glib2-devel  libibumad-devel opensm-devel libibmad tk libibmad-devel tk-devel libibcommon libvma librdmacm openssl-devel ibutils infiniband-diags


# note, I am adding some of these after the fact, they may not be able to be installed yet. Install them as you can otherwise

# 7z, a file decompression utility (because I like it, tar and unzip are also good)
dnf install -y p7zip p7zip-plugins

# If you're using a GUI, GUI for 7z: (commented to prevent mistakes such as installing Xorg)
#dnf install -y p7zip-gui

```

We need to fetch the drivers from OFED's page as well.

Go [here](http://downloads.openfabrics.org/OFED/ofed-4.8-1/) and then get the latest version. (Note, if a newer version has come out of 4.8.1, go up a folder and then get the newest version that is not daily AND is a greater number than your kernel version, found from running ```uname -a``` and reading the third field.)

```bash
# Download the RPM sources
wget http://downloads.openfabrics.org/OFED/ofed-4.8-1/OFED-4.8-1-rc2.tgz -O ofed.tgz

# Extract the files
7z x ofed.tgz
tar -xvf ofed.tar

# Optionally, remove the archives
rm ofed.t*

# cd into the new folder.
cd ofed..somepath../SRPMS

# Create the sources for installing from.
rpm -ihv *.rpm

# move to build directory
cd ~/rpmbuild/SPECS

# Run ls here. You should find a bunch of .spec files. If not, something is wrong.
ls

# This next stuff will take much longer than everything else, since it's compiling all of the packages from source.

# Install the necessary requirements (in this order)
rpmbuild -ba ~/rpmbuild/SPECS/libfabric.spec
rpmbuild -ba ~/rpmbuild/SPECS/libibscif.spec
rpmbuild -ba ~/rpmbuild/SPECS/infiniband-diags.spec
rpmbuild -ba ~/rpmbuild/SPECS/opensm.spec
rpmbuild -ba ~/rpmbuild/SPECS/ibpd.spec
rpmbuild -ba ~/rpmbuild/SPECS/dapl.spec

# Throw a reboot in here somewhere...

```

### Testing the capabiliites of IPoIB

To test the speed of the IB network from the IP layer, we will be using the program ```iperf```.

If you didn't install it already, do that now.

```
dnf install -y iperf
```
