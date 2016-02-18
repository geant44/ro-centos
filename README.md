steps to get a stateless read-only CentOS with a nfs-root.
=========================================================

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
