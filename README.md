# CentOS 7 Read-only cluster setup
## What is this?
This is a step-by-step how-to on how to setup a read-only CentOS 7 cluster. It is aimed at anyone wanting to setup a cluster with the following architecture:
  * A "master" server serving the OS to the other computers through an [NFS](https://en.wikipedia.org/wiki/Network_File_System) server for the files and [PXE](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) boot for the kernel/initial RAM disk.
  * Any number of clients of said server, who will have a read-only root system.

## Why would I want that?
There are two big advantages at using the preceding configuration, that can be useful in several situations. Firstly, having a read-only root gives you an increase in security and stability, giving you more peace of mind when exposing computers to the public (such as diskless stations in a library for example), or to code written by other people if you are running servers. The second advantage is that you only have to maintain/administrate "one" computer as all changes are reflected on all the clients. This makes processes such as updates, installation of software and so on much simpler. This setup also gives you the possibilty to run the clients diskless. Interested? Read on!

## Table of Contents:
1. [Planning your cluster](#planning-your-cluster)
  1. [General information](#general-information)
  2. [Diskless or Diskful setup](#diskless-or-diskful-setup)
2. [Installing the master server](#installing-the-master-server)
  1. [Installing Centos 7](#installing-centos-7)
  2. [PXE Server](#pxe-server)
  3. [NFS Server](#nfs-server)
3. [Creating the client image](#creating-the-client-image)
  1. [Setup the disks](#setup-the-disks)
  2. [Install necessary packages](#install-necessary-packages)
  3. [Install optional packages](#install-optional-packages)
  4. [Systemd units](#systemd-units)
  5. [Configuration files](#configuration-files)
    1. [Main configuration file](#main-configuration-file)
    2. [State and writable files/directories](#state-and-writable-files/directories)
  6. [Create the initrd](#create-the-initrd)
4. [Pushing the image to the server](#pushing-the-image-to-the-server)
  1. [Rsync the files](#rsync-the-files)
  2. [Put the final touches](#put-the-final-touches)
5. [Final tips and remarks](#final-tips-and-remarks)
  1. [Caveats](#caveats)
  2. [Helpful debugging tips](#helpful-debugging-tips)
  3. [Conclusion](#conclusion)


**WORK IN PROGRESS**
## Planning your cluster
### General information
The setup is quite flexible. You can have as many clients as you want, the only requirement are for them to be able to access the server over a (safe) network. Because the server can serve an arbitrarily large number of clients (depending only of your network connection, but as the server only has to serve the system once, it usually does not have to be able to support a heavy load), you can start even with only two machines, and then add more to your network according to your needs. Before we start, be sure to think about your network setup: you will to know the subnets on which your clients will be to configure the NFS server.

### Diskless or diskful setup
You will want to choose now if you want your clients to be diskless (Do not store anything on the client, allowing it to have no disk, or use it's disk for other data) or not (store client specific state information on the client's hard drive and not on the server). The two cases will be referred to as a "diskless" and "diskful" setups. Go for a diskless setup for added security and reduced costs, but at the price of being *strictly* stateless. Go for diskful if you want only the OS to be stateless, but want people to be able to store their work and specific configuration options.

Once you know which setup you want, and have the machines (or VMs if you just want to play around) ready, let's start. We will first install the master server, then use one of the clients to create the system that we will deploy. Finally we will push that system to the server, and talk about tips and what to do if things fail.

## Installing the master server
This machine is going to allow the clients to do networking boot using PXE and then serve them their system using NFS. The server's setup is distribution independant, but for simplicity CentOS 7 will also be used. First install a base CentOS 7 system, using whatever guide you wish. Once you are satisfied with you setup, come back to this guide.

### PXE server
_TODO_

### NFS server
You will need a functional NFS file server on this machine to serve the system files to the clients. First create the directories where you will store the client's data:
`master:~# mkdir -p /data/cluster/install`. Then, install the nfs server by installing the following packages: `master:~# yum install nfs-utils nfs-utils-lib`.
Configuration of which directories we want NFS to export is done in /etc/exports, so you will want to edit that file:
`master:~# vi /etc/exports`:
```
/data/cluster/install <ip range of clients>
```

If you are running a diskless setup, you will also want to create directories to store each client's state. These directories must all be under the same top level directory and should be named after each client's FQDN (the clients will receive their hostnames from the network): `master:~# mkdir /data/cluster/state/client{1,2,...,n}.mytest.local`

**OLD VERSION FOR REFERENCE**

Create base image 
-----------------
### Disks
First install CentOS to a machine that will be used to generate your base image. There are two main cases:
  * You wish to use the machine's disk, and you will need two partitions ondisk labeled following the RW\_LABAEL and STATE\_LABEL in your /etc/sysconfig/readonly-root file (stateless-rw and stateless-state by default).
  * You wish the system to act as if it were diskless (such as being really diskless, or using the disks for other data than the OS). In this case you will need to also setup your NFS server accordingly. We will cover this case with the setup of the NFS server.

### Packages
Install as usual, then ensure that the following packages are installed:
  * dracut-network
  * dracut-config-generic (needed to create non-host-specific images)
  * nfs-utils
  * rsync
Also install now any packages you will need later as the system will be read-only. (You can always correct errors later using ```mount -o remount,rw``` as long as you don't define your nfs shares as RO).

### Systemd units
You will want to disable
  * systemd-readahead-collect.service: It attempts to write the disk usage pattern used at boot time. See man systemd-readahead for more details. You can also add /.readahead in the rw or state lists if you wish to keep this function.

Depending on your setup, you need to test and watch with the following units:
  * systemd-tmpfiles: This service reads from /etc/tmpfiles.d,/usr/lib/tmpfiles.d and /run/tmpfiles.d to find conf files that require it to create/destroy temporary files/directories. Therefore any systemd-aware program may write in one of these directories a .conf file that asks for a particular directory/file to be created and/or removed on start/stop.  
  * daemons that need to read/write sockets in weird places (e.g. gsspi).
### Setup rw/state
Depending on your needs, you will need to add different files/dirs to the /etc/rwtab and /etc/statetab, respectively temporary and persistent places to write. The script will then take care of bound-mounting everything where they should be. Whatever your setup, you will want to make sure you have:
  * in /etc/rwtab:
    * /etc/localtime
    * /var/lock
    * /var/lib/postfix/master.lock
    * /var/lib/gssproxy
  * in /etc/statetab
    * /var/log (you might as well want to log to your master server's syslog though)

### Setup as read-only
To tell the system to boot in ro-mode with all the done config, you need to edit the /etc/sysconfig/readonly-root file:
  ```
  cat >>/etc/sysconfig/readonly-root <<EOF
  READONLY=yes
  TEMPORARY_STATE=yes
  RW\_MOUNT=/var/lib/stateless/writable
  RW\_LABEL=stateless-rw #/!\
  RW\_OPTIONS=
  STATE\_LABEL=stateless-state #/!\
  STATE\_MOUNT=/var/lib/stateless/state
  STATE\_OPTIONS=
  CLIENTSTATE= #/!\
  SLAVE\_MOUNTS=yes
  
  EOF
  ```
Depending on your case as explained in the Disks section, you will want to:
  * if you wish a diskless system, comment out the {RW,STATE}\_LABEL lines, and add <nfs server ip add>:/path/to/state/export (see the following nfs setup) in the CLIENTSTATE field.
  * if you wish to use the machine's disk, leave the labels uncommented and set to the labels you gave your partitions in the Disks part. Leave the CLIENTSTATE field empty.

### Modules
still need to untangle nfs mystery. For now, run ```modprobe nfs nfsv4```

### Setup nfs server
You will need to setup a server with an nfs export that will be used as the root for your stateless machines. It should not be necessary to force the export to be ro, though you could do it for security enhancement (note that you will then not be able to modify your system from one of the nodes using ```mount -o remount,rw``` and will either have to have a master with write access, or chroot from your server.)
If you also wish to keep your machine states in the nfs server, create another nfs export, the path of which goes in the CLIENTSTATE field in /etc/sysconfig/readonly-root. Inside the top directory of this export, create a folder containing the FQDN of each node you will deploy.

### create the initrd
#### first image generation
create a list of loaded kernel modules using the following command:
```xs=$(lsmod|grep -v Module|awk '{printf "%s ",$1}')``` (do not hesitate to check echo $xs to be sure you have not forgotten anything)
/*More dracut weird crap*/
Modify the /etc/dracut.conf file to add your network driver to the 'add\_drivers' field, and 'nfs' in the 'add\_dracutmodules' field.
Finally create your image using ```dracut -a "nfs network kernel-modules base" --add-drivers "$xs" --kernel-cmdline "root=nfs:<nfs-server ip>:/path/to/export/root" -f initramfs-nfs.img```
#### general setup
You are now going to want to setup the ```/etc/dracut.conf``` file so that a correct image is generated on kernel updates. You just need to hardcode in it the different cmdlines used previously (the ```kernel_cmdline``` option does not appear in the defaul example file but it is present -- see man dracut.conf). In the last part we will setup a script that automatically updates the nfs server.

### Put all files in the remote
copy the new initramfs to your remote PXEboot server
```scp initramfs-nfs.img root@<your server>:/your/pxeboot/path/```

Make sure you do not have autofs working (or you will have an infinite mounting loop   * much fun!) and copy the system to your nfs server:
```rsync -aH --exclude=/{proc,sys,dev,media,mnt,tmp,home,run,srv}/  * / root@<your server>:/path/to/export/root```

### Cleanup on the remote files
##### ssh into your server and edit the following files in the root export:
Your fstab should contain:
```
none    /tmp        tmpfs   defaults   0 0
tmpfs   /dev/shm    tmpfs   defaults   0 0
sysfs   /sys        sysfs   defaults   0 0
proc    /proc       proc    defaults   0 0
```
  * Remove any yum old files: ``` rm /var/lib/rpm/__db* ``` (otherwise the systemd service that does so on boot will fail)
  * Ensure you only have the lo interface in /etc/sysconfig/network-scripts/ifcfg-* -- dracut will autogenerate the needed files.
  * Remove the ```/etc/hostname``` file, so nodes will take their hostname from the network (you need to have a working dhc ofc).
  * Any other file that may contain HW specific info
