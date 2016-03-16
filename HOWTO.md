#summary nfsroot HOWTO
#labels Featured

_This document pertains to nfsroot version 3_

## Introduction ##

The **nfsroot** RPM is installed into a root image which is served to
clients.  When the client boots, one of several methods is selected
to make this image usable as the basis for a root file system.
For example, if the _aufs_ method is selected, a union file system combines
a read-only NFS export with a read-write tmpfs file system on the client
to make a read-write file system where any changes on the client go to
the tmpfs.  If the _bind_ method is selected, key directories in the read-only
root file system are copied to a local tmpfs, then the writeable tmpfs copies
are bind-mounted over the read-only originals.

The server must provide DHCP, TFTP, and NFS service.  Management of these
services and their config files is beyond the scope of the **nfsroot**
package.

**nfsroot** is sensitive to OS distro verison.
Version 2 works with RHEL5 (CHAOS 4).
Version 3 works with RHEL6 (CHAOS 5).
If you want to use it on a different distro, porting is straightforward;
open a bug in the **nfsroot** issue tracker on google code with your changes.

## Client Boot Sequence ##

  1. C: BIOS ROM extension on network card broadcasts DHCP request
  1. S: DHCP server responds with info for C's MAC address
  1. C: tftp `pxelinux.0` and `pxelinux.cfg` from the `/boot` in the root image
  1. S: allow tftp
  1. C: pxelinux prompts for kernel selection, and boots default after no input
  1. C: tftp's`vmlinuz` and `initramfs` from `/boot`
  1. S: allow tftp
  1. C: initramfs broadcasts DHCP request, now in Linux environment
  1. S: DHCP server responds with info for C's MAC address
  1. C: initramfs mounts root image from server (normally read-only), switches into it, and execs `/etc/rc.nfsroot`.
  1. S: allow NFS mount
  1. C: `/etc/rc.nfsroot` (now in the NFS root image) makes the root file system at least partially writeable by running `/etc/rc.nfsroot.<method>`, for each method in the METHODS list in `/etc/sysconfig/nfsroot`.  The successful script execs `/etc/rc.nfsroot-init`.
  1. C: `/etc/rc.nfsroot-init` runs everything in `/etc/rc.nfsroot.d` (for example, your cfengine scripts), then execs `/sbin/init`.
  1. C: boots as usual

## Example ISC dhcpd.conf ##

In this example, root images are located in `/tftpboot/images` on the server.
A singe host named `webb` boots the `test` image.

```
ddns-update-style none;

option space pxelinux;
option pxelinux.magic      code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;

# Needed for dhcp 3.0.5
next-server 192.168.1.169;

shared-network eth0 {
        not authoritative;
        use-host-decl-names true;

        site-option-space "pxelinux";
        option pxelinux.magic f1:00:74:7e;
        if exists dhcp-parameter-request-list {
                option dhcp-parameter-request-list = concat(option dhcp-parameter-request-list,d0,d1,d2,d3);
        }
        option pxelinux.reboottime 30;

        subnet 192.168.1.0 netmask 255.255.255.0 {
                option subnet-mask      255.255.255.0;
                option routers          192.168.1.222;

                host webb {
                        filename "images/test/boot/pxelinux.0";
                        option root-path "192.168.1.169:/tftpboot/images/test";
                        option pxelinux.configfile "pxelinux.cfg";
                        option pxelinux.pathprefix "/images/test/boot/";
                        hardware ethernet 00:13:72:18:DF:9A;
                        fixed-address 192.168.1.194;
                }
        } # end 192.168.1.0/255.255.255.0

} # end eth0
```

In nfsroot version 3, nfs mount options can be appended to the root-path line, e.g.
```
root-path "192.168.1.169:/tftpboot/images/test,nfsvers=3,rw,nolock,actimeo=600,nocto,nosharecache";
```

If you are aware of how dracut-network is configured, nfsroot is using this behind the scenes, adding a config file to /etc/dracut.conf.d which limits the dracut modules to the minimum required for NFS and network, and installing a dracut module called xnfsroot which fakes kernel command line options root=dhcp and init=/etc/rc.nfsroot (see below).

## Configuration ##

The file `/etc/sysconfig/nfsroot` in the root image contains several tunable
parameters.  This file should be updated on the server as its content is sourced early by the rc.nfsroot scripts:
  * `METHODS` - If set, rc.nfsroot will try these methods (in order) to make root read-write.
  * `TMPFSMAX` - If set, rc.nfsroot-method mounts tmpfs with `-osize=$TMPFSMAX`.  Otherwise the default is half of RAM.
  * `RAMDIRS` - If set, rc.nfsroot-bind will make working copies of these dirs on ramdisk.
  * `INITPROG` - If set, rc.nfsroot will exec this (with `$@` args) in place of `/sbin/init`.
  * `KDUMP_DIR` - Override default of `<tftpserver>:/var/crash`.
  * `KDUMP_DIR_MOUNTOPTS` - Set NFS mount options for KDUMP\_DIR.
  * `KDUMP_LEVEL` - Set vmcore filter mask (see `makedumpfile -d` option).
  * `KDUMP_FAILSAFE` - If vmcore save fails, `reboot` means reboot, `shell` means spawn a shell.

Kernel command line options can be changed in `/boot/pxelinux.cfg` in the root image (from the server).  The default linux entry is copied whenever a new kernel is added to the image so any options changed there will propagate to new entries as they are added.

## Kernel Updates ##

Nfsroot hooks into grubby so when a new kernel is installed/removed/updated, /boot content gets updated appropriately.

The main scripts are `/usr/sbin/configpxe` which manages pxelinux.cfg entries, and `/usr/sbin/nfsroot-setdefault` which manages /boot symlinks for vmlinuz and initramfs corresponding to the default pxelinux.cfg boot entries.
The scripts are called by grubby's `/sbin/new-kernel-pkg` via the following hooks: `/etc/kernel/postinst.d/nfsroot-postinst` and
`/etc/kernel/prerm.d/nfsroot-prerm`.

Two other helper scripts are `/usr/sbin/nfsroot-rebuild` which rebuilds all initramfs images upon an nfsroot update, or when manually invoked; and `/usr/sbin/nfsroot-kdumplinks` which maintains symbolic links required by kdump.

It is possible in version 3 to install multiple kernels, CHAOS or Red Hat, and have them available as pxelinux boot options simultaneously, with all the /boot content managed automatically.

## Boot Methods ##

The `rc.nfsroot` startup script runs chrooted in the read-only root image.
It runs _boot method_ startup scripts in the order determined by the `METHODS`
variable in `/etc/sysconfig/nfsroot`.  The first one that succeeds causes system startup to proceed.

### kdump ###

**nfsroot** works with RHEL kdump.  `nfsroot-kdumplinks` (described above) maintains symlinks
for initramfs images such that `/etc/init.d/kdump` will arm
kexec with it during system startup.  The crash sequence is as follows:

  1. C: crashing kernel kexecs stored kernel/initramfs
  1. C: initramfs broadcasts DHCP request
  1. S: dhcp server responds with info for C's MAC address
  1. C: initramfs mounts root image from server (normally read-only), switches into it, and execs `/etc/rc.nfsroot`.
  1. S: allow NFS mount
  1. C: `/etc/rc.nfsroot` runs `/etc/rc.nfsroot-kdump` which detects kdump environment
  1. C: `/etc/rc.nfsroot-kdump mounts KDUMP\_PATH on /var/crash
  1. S: allow NFS mount
  1. C: copy vmcore to /var/crash
  1. C: hard reboot

Vmcores can be examined with the crash utility, e.g.

```
crash <System.map> <vmlinux> /var/dumps/vmcore-ilc13-2007-11-04-19:12:21
crash> bt
```

Note that the file `/etc/kdump.conf` only affects the kdump initrd,
thus when **nfsroot** is providing the initramfs, that file has no effect.
However, `/etc/sysconfig/kdump` is still used by `/etc/init.d/kdump`.

### none ###

If the initial root file system is mounted read-write, the none method will detect this and declare victory.
This presumes that the root image is not shared.

### aufs ###

A ramdisk is created and root is crafted as a union file system
based on a read-only NFS branch, and a read-write ramdisk branch.
The aufs file system is **not** included in the CHAOS 5 kernel, so this module is historical.

### unionfs ###

Same as aufs but use the unionfs file system.
The unionfs file system is **not** included in the CHAOS 5 kernel, so this module is historical.

### bind ###

A ramdisk is created and directories listed in the `$RAMDIRS` variable
in `/etc/sysconfig/nfsroot` are copied from the read-only root to the
ramdisk, and bind-mounted on top of the original.

### rbind ###

Same as bind except root is a ramdisk, `$RAMDIRS` directories
are copied there and remaining top-level read-only directories are
bind-mounted there.

### ram ###

Similar to rbind except the entire root file system is copied
to the ramdisk.  Ensure that `$TMPFSMAX` is set to accomodate it.
This is really only practical for stripped-down appliance images.

## Notes and Caveats ##

With any of the methods, it is not recommended that the NFS
file system be changed on
the server while clients have it mounted.  A good strategy is
to have the root-path mentioned in dhcpd.conf be a symbolic link, then updates can be made by
rsyncing the current image to a new directory, changing the new image,
then updating the symlink to switch clients to the new image on the
next reboot.  Running clients will continue to use the old root image.

The NFS clients generate a lot of traffic revalidating cached data.
To address this, NFS mount options such as `actimeo=600,nocto` can be
added to retain cached data longer.  This costs nothing if, as suggested,
the read-only root file system will never change out from under the client.

You may wish to disable cron on diskless clients, or prune default crontabs
such as makewhatis, logwatch, mlocate, and rpm and instead run these as
part of an image update procedure.  Running them on the node increases
ramdisk utilization.

`pxelinux.cfg` may be kept under version control.  The "master" file
should contain only the global pxelinux options plus the default "linux"
boot label options.  After installing the master, run `nfsroot-rebuild`
(on the server!) to recreate entries for all the installed kernels.
A caveat is that all entries must have the same kernel command line arguments.

LFS says that `/var/tmp` state must persist across a reboot, while `/tmp`
need not persist.  On diskless systems, `/tmp` can be a ramdisk while `/var/tmp`
should be an NFS mount unique to the node.  Scripts that need to pass state
across a reboot should be wired for `/var/tmp`.

If a root image is exported read-write but configured to be mounted read-only
by **nfsroot**, `rc.nfsroot` halts boot-up because `/etc/rc.sysinit` will
attempt to remount root read-write.
Booting a read-write root image shared between clients may corrupt the image.

**kexec-tools** will test if `/etc/kdump.conf` is newer than the
initramfs image and attempt to rebuild the initrd if so.  This rebuild will fail or produce a broken initrd in the nfsroot
environment, so a test for this condition has been added to `rc.nfsroot-kdump`
to halt boot-up if found during boot.

## Resources ##

[Yum package manager home page](http://yum.baseurl.org/).

[The Syslinux Project home page](http://syslinux.zytor.com/wiki/index.php/The_Syslinux_Project).

[Internet Systems Consortium DHCP project page](http://www.isc.org/software/dhcp).

[Kdump, a Kexec-based Kernel Crash Dumping Mechanism](http://lse.sourceforge.net/kdump/documentation/ols2oo5-kdump-paper.pdf).

[Linux NFS-HOWTO](http://nfs.sourceforge.net/nfs-howto/)

[AUFS sourceforge project](http://aufs.sourceforge.net/)

[Unionfs: A Stackable Unification File System](http://www.fsl.cs.sunysb.edu/project-unionfs.html)