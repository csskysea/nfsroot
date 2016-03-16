#summary Using nfsroot in a Kernel Development Environment

## Introduction ##

An nfsroot based diskless system is an excellent platform for linux
kernel development because the root file system is protected from
damage from repeated crashes, and reboot time is relatively fast.
Because nfsroot is closely integrated with the Red Hat kernel RPM,
it can be tricky to set up a system for kernel testing.

## Building and Installing a Vanilla Kernel ##

The main challenge is that the redhat kernel includes hooks to rebuild
the initramfs for a new kernel, to create entries in pxelinux.cfg, to
run depmod, etc..  You can just run the appropriate scripts manually
after installing a kernel.

Avoid installing multiple kernels into an image.  There's a potential
for conflicts in /lib/firmware and it slows everything down.
Get rid of the redhat/chaos one that probably was installed when your
image was built using `rpm -e --nodeps kernel` (chrooted in the image
on the server of course).

When configuring your kernel, make sure that modules for the network device
you plan to boot from is getting built.

```
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
Initialized empty Git repository in /home/garlick/work/linux/.git/
remote: Counting objects: 2339358, done.
remote: Compressing objects: 100% (360172/360172), done.
Receiving objects: 100% (2339358/2339358), 466.84 MiB | 7.75 MiB/s, done.
remote: Total 2339358 (delta 1956888), reused 2339080 (delta 1956647)
Resolving deltas: 100% (1956888/1956888), done.
$ cd linux
$ make defconfig
$ vi .config
$ make oldconfig
$ make rpm
...
Wrote: /home/garlick/rpmbuild/SRPMS/kernel-3.3.0_rc5+-1.src.rpm
Wrote: /home/garlick/rpmbuild/RPMS/x86_64/kernel-3.3.0_rc5+-1.x86_64.rpm
Wrote: /home/garlick/rpmbuild/RPMS/x86_64/kernel-headers-3.3.0_rc5+-1.x86_64.rpm
...
$ cd /tftpboot/images/clarence
$ sudo chroot $(pwd) /usr/sbin/depmod -a 3.3.0-rc5+
$ sudo rpm -r $(pwd) -Uvh /home/garlick/rpmbuild/RPMS/x86_64/kernel-3.3.0_rc5+-1.x86_64.rpm
Preparing...                ########################################### [100%]
   1:kernel                 ########################################### [100%]
$ sudo chroot $(pwd) /usr/sbin/nfsroot-rebuild
nfsroot-rebuild: rebuilding 3.3.0-rc5+ initramfs
nfsroot-setdefault: DEFAULT set to 3.3.0-rc5+
$
```

Forget about crash dumps, kdump is a redhat thing.

## Other Neat Things to Do ##

Set up powerman and conman to control your test system.
Boot with serial consoles enabled.
Enable sysreq - you can send them on the serial console via conman
with
```
&B?
```
where ? is the sysreq character you want to send.