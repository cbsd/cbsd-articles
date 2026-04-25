# NFSv4/DHCP/PXE jail-based (vnet) service via (dnsmasq)

This article describes a way to run and use a container adapted to work with CBSD using [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html), which will provide (all together or optionally):

- DHCP service for virtual machines and containers using a virtual stack (e.g. vnet/VIMAGE);
- PXE service for running diskless OS (both virtual machines and bare metal);
- HTTP/NFSv4 service for distributing content to diskless OS;

During the setup process, we'll explore practical options for unattended/preseed installations and generating FreeBSD-based systems, as well as automated installation of several Linux distributions, implementing something similar to iVentoy. For example, the described method generates cloud images of guest systems for the CBSD project in a fully automated manner;

<center><img src="https://convectix.com/img/dnsmasq_cbsd1.png" width="1024" title="dnsmasq_pxe" alt="dnsmasq_pxe"/></center>


## Launching and configuring a container

To configure it correctly, you must first select the IP address (in this example, <strong>172.16.0.101</strong>) of the routed network (in this example: <strong>172.16.0.0/24</strong>), which you will assign to the DHCP/PXE service. Also, determine the address range for which the DHCP server will be responsible.

:bangbang: | :information_source: Important! This article uses file and directory paths to /usr/jails/*. If your CBSD working directory path is different, replace /usr/jails with your own.
:---: | :---

:bangbang: | :information_source: To simplify the setup and avoid overcomplication, we won't consider a scenario where the static and DHCP address ranges overlap or coincide. For example, suppose we have a subnet of 172.16.0.0/24. Let's assume the address range 172.16.0.2-99 is served by DHCP, and 172.16.0.101-243 is used for static addresses. Although specifying 172.16.0.1-254 for both static and DHCP addresses is possible, it requires the use of the CBSD hook mechanism to ensure that static addresses are excluded from DHCP distribution (and vice versa—exporting the list of addresses issued by the DHCP service as an exclusion list for static IPs).
:---: | :---


a) Run `cbsd initenv-tui` and set the `nodeippool` setting to 172.16.0.102-243 (we assigned 172.16.0.101 to dnsmasq). This will force CBSD to automatically assign static addresses for VMs and containers only within the specified range, ensuring they don't conflict with DHCP.

b) You can get a pre-built image from the repository, or use [CBSDFile](https://github.com/cbsd/cbsdfile-recipes/blob/master/jail/dnsmasq/CBSDfile) from the recipe collection (requires `git` to be installed) to generate a container from scratch. 

<details>
  <summary>Option 1: A pre-built dnsmasq image</summary>

WIP

</details>

<details>
  <summary>Option 2: Generate a dnsmasq container</summary>

```
git clone https://github.com/cbsd/cbsdfile-recipes.git /tmp/cbsdfile-recipes
cd /tmp/cbsdfile-recipes/jail/dnsmasq
```
We launch a container with the IP address <strong>172.16.0.101</strong>:
```
cbsd up ip4_addr=172.16.0.101
```
</details>

c) Let's launch the service reconfigurator:
```
cbsd jconfig jname=dnsmasq net
```
or in TUI version:
```
cbsd jconfig jname=dnsmasq net-tui
```

Example answers:
```
Trusted NFSv3/v4 network: 0
Please set DHCP-range: 172.16.0.2,172.16.0.99,12h
Please set GW4: 172.16.0.1
NS1: 172.16.0.1
NS2: 0
```

For non-interactive reconfiguration, use env or directly in CBSDFile (see the commented example in CBSDfile):
```
	export H_NFSNET="0" \
		H_IP4RANGE="172.16.0.2,172.16.0.100,12h" \
		H_GW4="172.16.0.1" \
		H_NS1="172.16.0.1" \
		H_NS2="0"
```

- <strong>H_NFSNET</strong> or <strong>NFSv3/v4 network</strong>: IP network in CIDR format (net/prefix) from which NFS resources (PXE/diskless) can be mounted (valid IP/network, or '0' if the NFS service is not enabled). We'll look at how to use this service below;
- <strong>H_IP4RANGE</strong> or <strong>DHCP-range</strong>: IP address range served (will be issued) by the DHCP server in the format <start>,<end>,<ttl>, for example: 10.0.0.5,10.0.0.10,12h - addresses from 10.0.0.5 to 10.0.0.10 will be issued for 12 hours;
- <strong>H_GW4</strong> or <strong>GW4</strong>: gateway address that the DHCP server will provide as the default route (valid gateway IP address);
- <strong>H_NS1</strong> or <strong>NS1</strong>: nameserver1, which the DHCP server will provide as the default route (valid DNS server IP address, or '0' if not provided);
- <strong>H_NS2</strong> or <strong>NS2</strong>: nameserver2, which the DHCP server will provide as the default route (valid DNS server IP address, or '0' if not provided);

After launching and initializing the container, our working IPXE file is ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe (if going from the host file system) or /usr/local/srv/pxe/main.ipxe relative to the FS root directly inside the container.

### Dump of leases file records

Accessible via HTTP request to the container's address, for example: <strong>http://172.16.0.101</strong>

<center><img src="https://convectix.com/img/dnsmasq_cbsd2.png" width="1024" title="dnsmasq_pxe" alt="dnsmasq_pxe"/></center>

Or via the CLI:
```
cbsd jexec jname=dnsmasq dnsmasq_records
```

<center><img src="https://convectix.com/img/dnsmasq_cbsd3.png" width="1024" title="dnsmasq_pxe" alt="dnsmasq_pxe"/></center>


## PXE/NFS(TFTP) service for Diskless systems and Unattended installations.

Let's look at several options for adding a PXE/diskless OS.

:bangbang: | :information_source: The ISO/PXE structure generation examples use the CBSD script `jail2iso`. If you use alternative mfsbsd generation mechanisms, skip the lines with 'jail2iso' (generating using your own method). However, the rest of the algorithm/entries/settings/tips will likely be useful and suitable for everyone.
:---: | :---

--

### FreeBSD PXE: unattended/preseed/auto install (using the official ISO image)

<details>
  <summary>Step-by-step HOW-TO</summary>

At the time of writing (2026), [bsdinstall(8)](https://man.freebsd.org/cgi/man.cgi?query=bsdinstall) supports preset parameters (see the SCRIPTING section) via the /etc/installerconfig file. However, the author is unaware of a way to pass a preseed configuration source (e.g., the URL of an external HTTP service) via kenv/loader without modifying the official ISO image. Note: There was a [GSoC](https://wiki.freebsd.org/SummerOfCode2014/FreeBSD_PXE_preseed) project to improve preseed installation support, but the work did not make it into the upstream distribution.

Therefore, we'll regenerate the ISO image to place the preseed file in the image's data structure and simultaneously convert the official image to MFSBSD mode (i.e., it will be fully loaded into RAM. In this case, the booted Live environment will be unaffected by potential network failures or closing the browser through which you present the ISO image to KVM/IPMI/Redfish).

In this example, we're working with FreeBSD 15.0-RELEASE.

1) Obtain the official image using the fetch utility or the CBSD profile:

```
fetch -o /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso.xz https://download.freebsd.org/releases/ISO-IMAGES/15.0/FreeBSD-15.0-RELEASE-amd64-disc1.iso.xz
```
or (+ scanning and searching for fast mirrors):
```
cbsd fetch_iso name=FreeBSD-15.0-x86_64 fastscan=1 keepname=1 dstdir=/tmp
```
and unpack:
```
xz -d /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso.xz
```

Result: presence of ISO image /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso, check:
```
file -s /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso
```
> /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso: ISO 9660 CD-ROM filesystem data (DOS/MBR boot sector) '15_0_RELEASE_AMD64_CD' (bootable)

2) Mount the contents of the ISO image to the /mnt directory (mdconfig + mount):
```
mount_cd9660 /dev/`mdconfig -a -t vnode -f /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso` /mnt
```

If the image is correct, we can list the image directories:
```
ls -1 /mnt/
```
```
.rr_moved
COPYRIGHT
bin
boot
dev
etc
lib
libexec
media
mnt
net
proc
rescue
root
sbin
tmp
usr
var
```

3) CBSD allows you to generate an ISO/memstick/MFSBSD image and/or PXE structure from a jail—we'll use this mechanism. To do this, create an empty jail and copy the ISO image contents from the /mnt directory into it. Optionally, you can immediately assign static IP addresses and enable SSH access (see cbsd jail2iso --help ), but in this example, we'll use DHCP configuration (we didn't run the dnsmasq container for nothing!).

a)
```
cbsd jcreate jname=myjail baserw=1 ver=empty pkg_bootstrap=0 floatresolv=0 applytpl=0 etcupdate_init=0
```

b) copy /mnt/* to the container directory:
```
cp -a /mnt/* ~cbsd/jails-data/myjail-data/
umount /mnt
```

We delete the entries in the original fstab, since they try to use CD9660/ISO:
```
truncate -s0 ~cbsd/jails-data/myjail-data/etc/fstab
```

Let's remove the unnecessary symbolic link /etc/resolv.conf and add our own NS for correct name resolution (by default, ISO has such a link):
```
lrw-r--r--   1 root wheel       31 Nov 28 08:18 resolv.conf -> /tmp/bsdinstall_etc/resolv.conf
```

```
rm -f ~cbsd/jails-data/myjail-data/etc/resolv.conf
echo "nameserver 8.8.8.8" > ~cbsd/jails-data/myjail-data/etc/resolv.conf
```

Let's add some SCRIPTING/preseed/unattended for the installer, ~cbsd/jails-data/myjail-data/etc/installerconfig:
```
#PARTITIONS=DEFAULT
#export DISK=$(sysctl -n kern.disks | awk '{print $1}')
export DISK="vtbd0"
export PARTITIONS="${DISK} GPT { 512K freebsd-boot, auto freebsd-ufs / }"

export DISTRIBUTIONS="kernel.txz base.txz"
export BSDINSTALL_SKIP_FIRMWARE=true
export BSDINSTALL_SKIP_HARDENING=true
export BSDINSTALL_SKIP_KEYMAP=true
export BSDINSTALL_SKIP_MANUAL=true
export BSDINSTALL_SKIP_SERVICES=true
export BSDINSTALL_SKIP_TIME=true
export BSDINSTALL_SKIP_USERS=true
export BSDINSTALL_SKIP_FINALCONFIG=true

# doesn't work
#export ROOTPASS_ENC='$6$hIzCCQNYT7wL0bpY$78hF0uQQRg45wzybpwx/lXCaX5mcKv2FW40cBG4xTQ28hAPv4ptcpz9WthWLGGgIi7Gi.Gd2qBjgxFbUMCxa10'
#export ROOTPASS_PLAIN="test"

export BSDINSTALL_DISTSITE="http://ftp.freebsd.org/pub/FreeBSD/releases/amd64/amd64/15.0-RELEASE"
export NONINTERACTIVE=yes


#!/bin/sh
sysrc ifconfig_DEFAULT=DHCP sshd_enable=YES sshd_flags="-oUseDNS=no -oPermitRootLogin=yes"

```


c) Generate MFSBSD. The `mfs_struct_only=1` parameter means we don't need to generate an image—only the MFSBSD file structure for the PXE service.

Make sure the roothack module is installed (required for MFSBSD):
```
cbsd module mode=install roothack
make -C /usr/local/cbsd/modules/roothack.d
```

```
cbsd jail2iso jname=myjail dstdir=/usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-15.0/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 kernel_dir=/usr/jails/jails-data/myjail-data/boot/kernel mfsbsd_ip_addr=REALDHCP mfsbsd_interface=auto
```

4) Add the menu item and relevant entries to the IPXE file ( ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe ):

```
item --key 1 FreeBSD-15.0         FreeBSD 15.0


:FreeBSD-15.0
set img FreeBSD-15.0
chain /${img}/boot/loader.efi dhcp.root-path=tftp:/${img} || goto failed
boot || goto failed
```
Additionally (but not necessarily), it makes sense to compress the kernel and modules—the loader first tries to use *.gz when loading files. This will significantly speed up loading:
```
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-15.0/boot/kernel/kernel
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-15.0/boot/kernel/*.ko
```

</details>

### FreeBSD PXE: Generating microvm for the bhyve hypervisor

<details>
  <summary>Step-by-step HOW-TO</summary>

:information_source: Note: This example is a custom version of https://github.com/cbsd/microbhyve, but uses PXE (only a few lines of kernel config and index files differ for minimalism).

The previous example used the official ISO image (which, however, required some modification). In this example, we'll create a virtual machine entirely from the CBSD jail, minimizing it as much as possible without resorting to hacks. This example can be considered a base for creating any minimalist firmware based on FreeBSD.

1)
Let's create an empty container:
```
cbsd jcreate jname=micro1 baserw=1 ver=empty pkg_bootstrap=0 floatresolv=0 applytpl=0 etcupdate_init=0
```

2) Let's fill the container structure with an index file that contains a minimal list of files (supplied with CBSD as an example of minimalism):


:bangbang: | :information_source: Note! We fill files based on the index, taking them from your host system (basedir=/) files (for example, password files will be taken from your host). You can use alternative files as a source, for example, ~cbsd/basejail/bases_XXX_YYY_VV/ directories
:---: | :---
```
cbsd copy-binlib basedir=/ chaselibs=1 dstdir=/usr/jails/jails-data/micro1-data filelist=/usr/local/cbsd/share/FreeBSD-microbhyve-pxe.txt.xz
```
3) Customization. Since we're creating a truly minimal image, it doesn't include a network configurator or dhclient, so we'll configure the network via /etc/rc.local – change the IP address and gateway to your working ones (instead of 172.16.0.99 and 172.16.0.1)!
```
cbsd sysrc jname=micro1 \
        sshd_flags="-oUseDNS=no -oPermitRootLogin=yes" \
        root_rw_mount="YES" \
        sshd_enable=YES \
        rc_startmsgs="YES" 

cat > /usr/jails/jails-data/micro1-data/etc/rc.local <<EOF
/sbin/ifconfig vtnet0 inet 172.16.0.99/24 up
/sbin/route add default 172.16.0.1
EOF

truncate -s0 /usr/jails/jails-data/micro1-data/etc/fstab
cp -a /etc/ssh /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/gss /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/pam.d /usr/jails/jails-data/micro1-data/etc/
mkdir -p /usr/jails/jails-data/micro1-data/var/empty /usr/jails/jails-data/micro1-data/var/log /usr/jails/jails-data/micro1-data/var/run /usr/jails/jails-data/micro1-data/root /usr/jails/jails-data/micro1-data/dev
chmod 0700 /usr/jails/jails-data/micro1-data/var/empty
pw -R /usr/jails/jails-data/micro1-data usermod root -s /bin/sh

# strip debug info via strip(1)
find /usr/jails/jails-data/micro1-data/ -type f -perm +111 -exec strip {} \;
```
4) Building the BHYVE-PXE kernel, a minimal working kernel config (see ~cbsd/etc/defaults/FreeBSD-kernel-BHYVE-PXE-amd64-15.0 ) for bhyve+PXE (supplied with CBSD as an example of minimalism):
```
cbsd srcup
cbsd kernel name=BHYVE-PXE
```
5) Make sure the roothack module is installed (required for MFSBSD):
```
cbsd module mode=install roothack
make -C /usr/local/cbsd/modules/roothack.d
```
generate PXE hierarchy:
```
cbsd jail2iso jname=micro1 dstdir=/usr/jails/jails/dnsmasq/usr/local/srv/pxe/MicroBSD-15.0/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXE
```
hint: if you want to generate a bootable image (in the /tmp directory) of the virtual machine:
```
cbsd jail2iso jname=micro1 dstdir=/tmp/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXE
```

edit /usr/jails/jails/dnsmasq/usr/local/srv/pxe/MicroBSD-15.0/boot/loader.conf to look like this:
```
if_vtnet_load="YES"

kernels_autodetect="NO"
kern.msgbuf_show_timestamp=1
hw.usb.no_boot_wait=1
hw.usb.no_shutdown_wait="1"
hw.usb.no_pf="1"
hw.memtest.tests=0
boot_serial=NO
boot_multicons=NO
console=efi

autoboot_delay="-1"

net.inet.ip.fw.default_to_accept="1"
vfs.mountroot.timeout=3
# MFS
mfs_load="YES"
mfs_name="/mfsroot"
mfs_type="mfs_root"
vfs.root.mountfrom="ufs:/dev/md0"
vfs.root_mount_always_wait=1
```

6) Add the menu item and relevant entries to the IPXE file ( ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe ):

```
item --key 2 MicroBSD-15.0         MicroBSD 15.0


:MicroBSD-15.0
set img MicroBSD-15.0
chain /${img}/boot/loader.efi dhcp.root-path=tftp:/${img} || goto failed
boot || goto failed
```

Additionally (but not necessarily), it makes sense to compress the kernel and modules—the loader first tries to use *.gz when loading files. This will significantly speed up loading:
```
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/Micro-15.0/boot/kernel/kernel
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/Micro-15.0/boot/kernel/*.ko
```

</details>

### FreeBSD PXE: A Diskless Workstation Using NFSv4

<details>
  <summary>Step-by-step HOW-TO</summary>

At the time of this writing, the official FreeBSD loader has extremely limited support for the network stack (see /usr/src/stand/libsa/* ) and NFS in particular (see: /usr/src/stand/libsa/nfs.c, /usr/src/stand/libsa/nfsv2.h )

Due to the fact that modern NFS servers do not recommend and eliminate UDP transport, and NFSD deployed in a jail vnet [does not support UDP](https://people.freebsd.org/~rmacklem/nfsd-vnet-prison-setup.txt)
> At this time, it is possible to enable NFSv3 and/or NFSv4 over TCP, but not NFSv3 over UDP

We'll again abandon the use of a vanilla FreeBSD image and attempt to use NFS with parameters in loader.conf (maybe this will work someday) like this:
```
#vfs.root_mount_always_wait=1
vfs.root.mountfrom="172.16.0.101:/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw"
vfs.root.mountfrom.options=rw

# Client IP address and mask
boot.netif.ip="172.16.0.X"
boot.netif.netmask="255.255.255.0"
boot.netif.gateway="172.16.0.1"
#boot.netif.hwaddr="XX:XX:XX:XX:XX:XX" # MAC address of your interface
#boot.netif.name="vtnet0"

# Данные NFS-сервера
boot.nfsroot.server="172.16.0.101"
boot.nfsroot.path="/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw"
```

We'll again use the MFSBSD layer, which will boot the system with a full network stack and NFSv4 support, after which we'll pivot/reroot to the full environment (which can be encrypted).

1) Prepare a directory for the full system (we'll assume you have the source code in /usr/src and have built the world using `make -C /usr/src buildworld`. You can install the database using base-in-pkg via `pkg`, `bsdinstall jail`, or simply copy the existing CBSD database from the ~cbsd/basejail/base_XXX_YYY_VER directory):
```
mkdir -p /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw
make -C /usr/src installworld distribution DESTDIR="/usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw"
```

Let's configure /etc/rc.conf within the NFS environment so that the system boots correctly:
```
touch ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw/etc/rc.conf
sysrc -qf ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw/etc/rc.conf nfs_client_enable="YES" nfs_access_cache="60" root_rw_mount="NO" varmfs="NO" tmpmfs="NO" ifconfig_DEFAULT="DHCP"
```

Let's configure /etc/fstab inside the NFS environment so that the system can mount NFS and tmpfs, create the file ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw/etc/fstab with the following contents:


```
172.16.0.101:/FreeBSD-NFS-15.0-rw	/	nfs	rw,noinet6,nfsv4,nolockd,tcp,rw,rsize=1048576,wsize=1048576,noatime	0 0
tmpfs	/tmp		tmpfs	rw	0 0
tmpfs	/var/run	tmpfs	rw	0 0

```

2) Let's make sure that /etc/exports inside the dnsmasq container contains the relevant entries for exporting the required directories: ~cbsd/jails/dnsmasq/etc/exports
```
/usr/local/srv/pxe -alldirs -network 172.16.0.0/24 -maproot=root -sec=sys
V4: /usr/local/srv/pxe -network 172.16.0.0/24
```

:bangbang: | :information_source: If dnsmasq is running on ZFS, you may need to set the sharenfs flag on this dataset: `zfs set sharenfs=on <dataset>`

Make sure the service restarts:
```
cbsd jexec jname=dnsmasq /etc/rc.d/mountd restart
```

3) Otherwise, our MFS image will be identical to the BHYVE-PXE version, except that we'll need NFS support in the kernel configuration, and in /etc/rc.d/ we'll place a script that mounts the NFS and executes the pivot_root/nfs function:

Building the BHYVE-PXENFS kernel. A minimal working kernel configuration (see ~cbsd/etc/defaults/FreeBSD-kernel--BHYVE-PXENFS-amd64-15.0 ) (supplied with CBSD as an example of minimalism):
```
cbsd srcup
cbsd kernel name=BHYVE-PXENFS
```

4) Let's create an empty container:
```
cbsd jcreate jname=micro1 baserw=1 ver=empty pkg_bootstrap=0 floatresolv=0 applytpl=0 etcupdate_init=0
```

5) Let's fill the container structure with an index file that contains a minimal list of files (supplied with CBSD as an example of minimalism):


:bangbang: | :information_source: Note! We fill files based on the index, taking them from your host system (basedir=/) files (for example, password files will be taken from your host). You can use alternative files as a source, for example, ~cbsd/basejail/bases_XXX_YYY_VV/ directories
:---: | :---
```
cbsd copy-binlib basedir=/ chaselibs=1 dstdir=/usr/jails/jails-data/micro1-data filelist=/usr/local/cbsd/share/FreeBSD-microbhyve-pxenfs.txt.xz
```
6) Customization. Since we're creating a truly minimal image, it doesn't include a network configurator or dhclient, so we'll configure the network via /etc/rc.local – change the IP address and gateway to your working ones (instead of 172.16.0.99 and 172.16.0.1)!
```
cbsd sysrc jname=micro1 \
	root_rw_mount="NO"

truncate -s0 /usr/jails/jails-data/micro1-data/etc/fstab
cp -a /etc/ssh /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/gss /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/pam.d /usr/jails/jails-data/micro1-data/etc/
mkdir -p /usr/jails/jails-data/micro1-data/var/empty /usr/jails/jails-data/micro1-data/var/log /usr/jails/jails-data/micro1-data/var/run /usr/jails/jails-data/micro1-data/root /usr/jails/jails-data/micro1-data/dev
chmod 0700 /usr/jails/jails-data/micro1-data/var/empty
pw -R /usr/jails/jails-data/micro1-data usermod root -s /bin/sh
find /usr/jails/jails-data/micro1-data/ -type f -perm +111 -exec strip {} \;
```

Create a file in ~cbsd/jails-data/micro1-data/etc/rc.local with the following contents:

```
/sbin/ifconfig vtnet0 inet 172.16.0.99/24 up
/sbin/route add default 172.16.0.1

getdesk_nfs=$( /bin/kenv getdesk_nfs )
getdesk_nfs_opts=$( /bin/kenv getdesk_nfs_opts )

if [ -z "${getdesk_nfs}" ]; then
	echo "error: no such getdesk_nfs kenv, checkout PXE server settings"
	exec /bin/sh
fi
if [ -z "${getdesk_nfs_opts}" ]; then
	echo "error: no such getdesk_nfs_opts kenv, checkout PXE server settings"
	exec /bin/sh
fi

# mount test:
[ ! -d /mnt ] && mkdir /mnt

for i in 1 2 3 4 5 6; do
	echo "mount [${i}/6]: ${getdesk_nfs} ..."
	set -o xtrace
	/bin/timeout 3 /sbin/mount -o ${getdesk_nfs_opts} ${getdesk_nfs} /mnt
	_ret=$?
	set +o xtrace
	if [ ${_ret} -eq 0 ]; then
		echo "   * mount ok"
		break
	fi
	sleep 1
done

if [ ${_ret} -ne 0 ]; then
	echo "mount failed: /sbin/mount -o ${getdesk_nfs_opts} ${getdesk_nfs} /mnt"
	exec /bin/sh
fi

/sbin/umount -f /mnt

echo "reroot -> nfs:${getdesk_nfs} ..."
/bin/kenv vfs.root.mountfrom="nfs:${getdesk_nfs}"
/bin/kenv vfs.root.mountfrom.options="${getdesk_nfs_opts}"
/sbin/reboot -r
```

7) Make sure the roothack module is installed (required for MFSBSD):
```
cbsd module mode=install roothack
make -C /usr/local/cbsd/modules/roothack.d
```
8) generate PXE hierarchy:
```
cbsd jail2iso jname=micro1 dstdir=/usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXENFS
```
hint: If you want to generate a bootable image (in the /tmp directory) of a virtual machine:
```
cbsd jail2iso jname=micro1 dstdir=/tmp/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXENFS
```

Add two lines to ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/boot/loader.conf to pass information about mounting NFS:
```
# see the contents of /etc/rc.local
getdesk_nfs="172.16.0.101:/FreeBSD-NFS-15.0-rw"
getdesk_nfs_opts="nfsv4,tcp,rw,rsize=1048576,wsize=1048576"
```

9) Add the menu item and relevant entries to the IPXE file ( ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe ):

```
item --key 2 FreeBSD-NFS-15.0         FreeBSD-NFS-15.0


:FreeBSD-NFS-15.0
set img FreeBSD-NFS-15.0
chain /${img}/boot/loader.efi dhcp.root-path=tftp:/${img} || goto failed
boot || goto failed
```

Additionally (but not necessarily), it makes sense to compress the kernel and modules—the loader first tries to use *.gz when loading files. This will significantly speed up loading:
```
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/boot/kernel/kernel
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/boot/kernel/*.ko
```

</details>

### NetBSD PXE: sysinst (Using the official ISO image)

<details>
  <summary>Step-by-step HOW-TO</summary>

Here’s an example of a NetBSD PXE installation using a dnsmasq container.

1) Fetch the official image using the fetch utility or via a CBSD profile:

```
fetch -o /tmp/NetBSD-10.1-amd64.iso https://cdn.netbsd.org/pub/NetBSD/NetBSD-10.1/images/NetBSD-10.1-amd64.iso
```
Alternatively (includes scanning and picking the fastest mirrors):

```
cbsd fetch_iso name=NetBSD-10-x86_64 fastscan=1 keepname=1 dstdir=/tmp
```

You should now have the ISO at /tmp/NetBSD-10.1-amd64.iso. Let's verify:

```
file -s /tmp/NetBSD-10.1-amd64.iso
```

> /tmp/NetBSD-10.1-amd64.iso: ISO 9660 CD-ROM filesystem data 'NETBSD_101' (bootable)

2) Mount the ISO to the /mnt directory (using mdconfig + mount):
```
mount_cd9660 /dev/`mdconfig -a -t vnode -f /tmp/NetBSD-10.1-amd64.iso` /mnt
```

If the image is valid, you'll be able to list its contents:
```
ls -1 /mnt/
```
```
.cshrc
.profile
.rr_moved
altroot
amd64
bin
boot
boot.cfg
dev
etc
install.sh
lib
libdata
libexec
mnt
mnt2
netbsd
rescue
root
sbin
stand
targetroot
tmp
usr
var
```

2) Setting up the PXE structure

a)
Copy everything from /mnt/* to the container's directory:
```
mkdir -p ~cbsd/jails/dnsmasq/usr/local/srv/pxe/NetBSD-10.1
cp -a /mnt/* ~cbsd/jails/dnsmasq/usr/local/srv/pxe/NetBSD-10.1/
umount /mnt
```

b) Configuring /usr/local/srv/pxe/boot.cfg (Hint: when specifying the root path in /usr/local/srv/pxe/NetBSD-10.1/, keep in mind it always looks relative to the TFTP root):
```
banner=Welcome to NetBSD Network Install
menu=Install NetBSD:boot net0:/NetBSD-10.1/amd64/binary/kernel/netbsd-INSTALL.gz -s
menu=Install NetBSD (auto-root):boot net0:/NetBSD-10.1/amd64/binary/kernel/netbsd-INSTALL.gz -a
timeout=15
default=1
```

c) The installer uses these default parameters; you'll need to update them manually during the interactive setup:

Host: cdn.NetBSD.org
Base directory: pub/NetBSD/NetBSD-10.1
Binary set directory: /amd64/binary/sets
Source set directory: /source/sets


</details>


### Debian PXE: unattended/preseed/auto install (using the official ISO image)

<details>
  <summary>Step-by-step HOW-TO</summary>

An example of an automated Debian installation using a preseed script and the PXE service in a dnsmasq container.

As with the FreeBSD ISO, we can't use the official ISO image alone, as it doesn't include PXE-compatible files ( initrd/vmlinuz ). We'll download them separately and place them in the PXE structure. Let's start with the official disk.

In this example, we're working with <strong>Debian 13.3.0 ( trixie ).

1) Obtain the official image using the `fetch` utility or the CBSD profile:

```
fetch -o /tmp/debian-13.3.0-amd64-DVD-1.iso https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/debian-13.3.0-amd64-DVD-1.iso
```
or (+ scanning and searching for fast mirrors):
```
cbsd fetch_iso name=Debian-13-x86_64 fastscan=1 keepname=1 dstdir=/tmp
```
Result: presence of ISO image /tmp/debian-13.3.0-amd64-DVD-1.iso, check:
```
file -s /tmp/debian-13.3.0-amd64-DVD-1.iso
```

> /tmp/debian-13.3.0-amd64-DVD-1.iso: ISO 9660 CD-ROM filesystem data (DOS/MBR boot sector) 'Debian 13.3.0 amd64 1' (bootable)

2) Mount the contents of the ISO image to the /mnt directory (mdconfig + mount):
```
mount_cd9660 /dev/`mdconfig -a -t vnode -f /tmp/debian-13.3.0-amd64-DVD-1.iso` /mnt
```

If the image is correct, we can list the image directories:
```
ls -1 /mnt/
```
```
.disk
EFI
README.html
README.mirrors.html
README.mirrors.txt
README.source
README.txt
boot
css
debian
dists
doc
firmware
install
install.amd
isolinux
md5sum.txt
pics
pool
```

2) Filling the PXE structure

a)
copy /mnt/* to the container directory:
```
mkdir -p ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Debian-13.3.0
cp -a /mnt/* ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Debian-13.3.0
umount /mnt
```

b) replace vmlinux/initrd with PXE/NFS support:
```
cd /tmp
fetch -o /tmp/netboot.tar.gz http://ftp.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/netboot.tar.gz
tar xfz netboot.tar.gz
mv /tmp/debian-installer ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Debian-13.3.0/
```

c) Let's create a d-i (Debian-installer) preseed file referenced by our download:
```
mkdir -p /usr/jails/jails/dnsmasq/usr/local/srv/debian13
```

<details>
  <summary>Contents of /usr/jails/jails/dnsmasq/usr/local/srv/debian13/auto.cfg:</summary>

```
d-i mirror/http/hostname string 172.16.0.101
d-i mirror/http/directory string /pxe/Debian-13.3.0
d-i debian-installer/allow_unauthenticated boolean true

d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i keyboard-configuration/xkb-keymap select us
d-i clock-setup/utc boolean true

d-i time/zone string Europe/Moscow
d-i clock-setup/ntp boolean true

d-i localechooser/supported-locales multiselect en_US.UTF-8, ru_RU.UTF-8

d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string space-controller
d-i netcfg/get_domain string local
d-i debian-installer/locale string en_US.UTF-8
d-i time/zone string RU/Moscow

d-i passwd/make-user boolean false
d-i passwd/root-login boolean true
d-i passwd/root-password password cbsd
d-i passwd/root-password-again password cbsd
d-i user-setup/allow-password-weak boolean true

d-i apt-setup/cdrom/set-first boolean false
d-i apt-setup/cdrom/set-next boolean false
d-i apt-setup/cdrom/set-failed boolean false

d-i partman/early_command string debconf-set partman-auto/disk "$(list-devices disk | grep -v nvme[0-9]c[0-9] | head -n1)"
d-i partman-auto/method string lvm
d-i partman-auto/choose_recipe select atomic
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/mount_style select uuid
d-i partman/confirm_nooverwrite boolean true

# Keep that one set to true so we end up with a UEFI enabled
# system. If set to false, /var/lib/partman/uefi_ignore will be touched
d-i partman-efi/non_efi_system boolean true

# enforce usage to GPT
d-i partman-basicfilesystems/no_swap boolean false
d-i partman-basicfilesystems/choose_label string gpt
d-i partman-basicfilesystems/default_label string gpt
d-i partman-partitioning/choose_label string gpt
d-i partman-partitioning/default_label string gpt
d-i partman/choose_label string gpt
d-i partman/default_label string gpt

d-i partman/mount_style select uuid
d-i partman-md/confirm boolean true

d-i apt-setup/use_mirror boolean false
d-i apt-setup/services-select multiselect
apt-cdrom-setup apt-setup/cdrom/set-first boolean false
popularity-contest popularity-contest/participate boolean false
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean true
tasksel tasksel/first multiselect standard

apt-cacher-ng apt-cacher-ng/tunnelenable boolean false

# pkg
d-i pkgsel/include string openssh-server

d-i finish-install/reboot_in_progress note

###
#d-i partman-auto-lvm/guided_size string 50%
d-i partman-auto-lvm/guided_size string max
d-i partman-auto-lvm/new_vg_name string sys_vg01

#d-i partman-auto/choose_recipe select boot-root
#d-i partman-auto/choose_recipe select atomic
d-i partman-auto/expert_recipe string                           \
        boot-root ::                                            \
                256 8 1024 free                                 \
                $iflabel{ gpt }                                 \
                $reusemethod{ }                                 \
                method{ efi }                                   \
                format{ }                                       \
                .                                               \
                512 16 512 ext4                                 \
                $primary{ } $bootable{ }                        \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ /boot }                             \
                .                                               \
                20000 64 -1 ext4                                \
                $lvmok{ }                                       \
                in_vg { sys_vg01 }                              \
                lv_name{ root }                                 \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ / }                                 \
                .

```

</details>
</details>

### Rocky/CentOS PXE: unattended/preseed/auto install (using the official ISO image)

<details>
  <summary>Step-by-step HOW-TO</summary>

An example of an automated Rocky installation using a preseed script and the PXE service in a dnsmasq container.

//As with the FreeBSD ISO, we can't use the official ISO image alone, as it doesn't include PXE-compatible files (initrd/vmlinuz). We'll download them separately and place them in the PXE structure. Let's start with the official disk.

In this example, we're working with <strong>Rocky 10.1</strong>.

1) Get the official image using the fetch utility or the CBSD profile:

```
fetch -o /tmp/Rocky-10.1-x86_64-dvd1.iso https://download.rockylinux.org/pub/rocky/10/isos/x86_64/Rocky-10.1-x86_64-dvd1.iso
```
or (+ scanning and searching for fast mirrors):
```
cbsd fetch_iso name=Rocky-10-x86_64 fastscan=1 keepname=1 dstdir=/tmp
```
Result: the ISO image /tmp/Rocky-10.1-x86_64-dvd1.iso is present, check:
```
file -s /tmp/Rocky-10.1-x86_64-dvd1.iso
```

> /tmp/Rocky-10.1-x86_64-dvd1.iso: ISO 9660 CD-ROM filesystem data (DOS/MBR boot sector) 'Rocky-10-1-x86_64-dvd' (bootable)

2) Mount the contents of the ISO image to the /mnt directory (mdconfig + mount):
```
mount_cd9660 /dev/`mdconfig -a -t vnode -f /tmp/Rocky-10.1-x86_64-dvd1.iso` /mnt
```

If the image is correct, we can list the image directories:
```
ls -1 /mnt/
```
```
.discinfo
.treeinfo
AppStream
BaseOS
COMMUNITY-CHARTER
EFI
EULA
LICENSE
RPM-GPG-KEY-Rocky-10
RPM-GPG-KEY-Rocky-10-Testing
boot
extra_files.json
images
media.repo
```

2) Populating the PXE structure

a)
Copy /mnt/* to the container directory:
```
mkdir -p ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Rocky-10.1
cp -a /mnt/* ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Rocky-10.1
umount /mnt
```

b) Let's create a kickstart preseed file that our download references:
```
mkdir -p /usr/jails/jails/dnsmasq/usr/local/srv/rocky10
```

<details>
  <summary>Contents of /usr/jails/jails/dnsmasq/usr/local/srv/rocky10/auto.ks:</summary>

```
d-i mirror/http/hostname string 172.16.0.101
d-i mirror/http/directory string /pxe/Debian-13.3.0
d-i debian-installer/allow_unauthenticated boolean true

d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i keyboard-configuration/xkb-keymap select us
d-i clock-setup/utc boolean true

d-i time/zone string Europe/Moscow
d-i clock-setup/ntp boolean true

d-i localechooser/supported-locales multiselect en_US.UTF-8, ru_RU.UTF-8

d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string space-controller
d-i netcfg/get_domain string local
d-i debian-installer/locale string en_US.UTF-8
d-i time/zone string RU/Moscow

d-i passwd/make-user boolean false
d-i passwd/root-login boolean true
d-i passwd/root-password password cbsd
d-i passwd/root-password-again password cbsd
d-i user-setup/allow-password-weak boolean true

d-i apt-setup/cdrom/set-first boolean false
d-i apt-setup/cdrom/set-next boolean false
d-i apt-setup/cdrom/set-failed boolean false

d-i partman/early_command string debconf-set partman-auto/disk "$(list-devices disk | grep -v nvme[0-9]c[0-9] | head -n1)"
d-i partman-auto/method string lvm
d-i partman-auto/choose_recipe select atomic
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/mount_style select uuid
d-i partman/confirm_nooverwrite boolean true

# Keep that one set to true so we end up with a UEFI enabled
# system. If set to false, /var/lib/partman/uefi_ignore will be touched
d-i partman-efi/non_efi_system boolean true

# enforce usage to GPT
d-i partman-basicfilesystems/no_swap boolean false
d-i partman-basicfilesystems/choose_label string gpt
d-i partman-basicfilesystems/default_label string gpt
d-i partman-partitioning/choose_label string gpt
d-i partman-partitioning/default_label string gpt
d-i partman/choose_label string gpt
d-i partman/default_label string gpt

d-i partman/mount_style select uuid
d-i partman-md/confirm boolean true

d-i apt-setup/use_mirror boolean false
d-i apt-setup/services-select multiselect
apt-cdrom-setup apt-setup/cdrom/set-first boolean false
popularity-contest popularity-contest/participate boolean false
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean true
tasksel tasksel/first multiselect standard

apt-cacher-ng apt-cacher-ng/tunnelenable boolean false

# pkg
d-i pkgsel/include string openssh-server

d-i finish-install/reboot_in_progress note

###
#d-i partman-auto-lvm/guided_size string 50%
d-i partman-auto-lvm/guided_size string max
d-i partman-auto-lvm/new_vg_name string sys_vg01

#d-i partman-auto/choose_recipe select boot-root
#d-i partman-auto/choose_recipe select atomic
d-i partman-auto/expert_recipe string                           \
        boot-root ::                                            \
                256 8 1024 free                                 \
                $iflabel{ gpt }                                 \
                $reusemethod{ }                                 \
                method{ efi }                                   \
                format{ }                                       \
                .                                               \
                512 16 512 ext4                                 \
                $primary{ } $bootable{ }                        \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ /boot }                             \
                .                                               \
                20000 64 -1 ext4                                \
                $lvmok{ }                                       \
                in_vg { sys_vg01 }                              \
                lv_name{ root }                                 \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ / }                                 \
                .

```

</details>
</details>


## Useful links
<details>
  <summary>Some useful resources related to the topic</summary>

(If you know more, feel free to add them)

https://docs.freebsd.org/en/books/handbook/advanced-networking/#network-diskless

https://www.iventoy.com/en/index.html

https://github.com/annetutil/annet

https://theforeman.org

https://developer.hashicorp.com/packer

https://fai-project.org/

https://www.yoctoproject.org/

</details>
