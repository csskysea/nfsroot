## Introduction ##

Sometimes BIOS and other firmware updates can only be applied using
a DOS update utility.  Starting with version 3.3, nfsroot provides a [FreeDOS](http://www.freedos.org)
boot image with serial console capability for your convenience.

In a cluster, a mass BIOS update could be orchestrated by driving
multiple serial consoles in parallel with [ConMan](http://code.google.com/p/conman)
or by adding the update script to `autoexec.bat`, after testing one interactively of
course!

Warning: some BIOS update programs invalidate the CMOS checksum.  The BIOS may
pause at the next boot with an interactive prompt to accept defaults.
Make sure your update procedure handles save/restore of CMOS or includes
reasonable defaults before proceeding with a mass update.

### FreeDOS Image Contents ###

The provided image is based on a FreeDOS 1.0 floppy boot image, expanded so that it has approximately 9MB free for your payload.  By default, only the himem driver is loaded.
An `autoexec.bat` redirects the console if the **sercons** kernel command line argument is present, and configures the serial port rate according to the **baud** or **baudhard** argument:

**sercons=com1** selects the port used for redirection (required)

**baud=value** sets the baud rate for values of 38,400 or less (optional)

**baudhard=value** sets the baud rate factor for higher baud rates (optional)

### Adding Payload ###

A custom image containing your application can be created by copying the provided image
and adding your payload using mtools or loopback mount.  For example:
```
$ mkdir x7sla
$ unzip -d x7sla ~/Downloads/x7sla0.513.zip
$ cp /boot/freedos.img myfreedos.img
$ export MTOOLSRC=mtoolsrc
$ echo 'drive y: file="myfreedos.img"' >mtoolsrc
$ mcopy -s x7sla y:
$ rm mtoolsrc
```

Copy your boot image into /boot and create a pxlinux.cfg entry for it:
```
label myfreedos
   kernel memdisk
   append nopassany=1 stack=2048 raw=1 initrd=myfreedos.img sercons=com1 baudhard=1152
```

### Example: AMI BIOS Update ###

```
boot: myfreedos
Loading memdisk..
Loading myfreedos.img......................................................................................................................................................................................ready.
```
Here FreeDOS waits 5 seconds for the user to choose a non-default configuration
via the VGA display and keyboard.  By default, only himem is loaded.
The autoexec.bat then redirects the console to the specified device.
```
A:\>dir
 Volume in drive A has no label
 Volume Serial Number is 05D8-5287
 Directory of A:\

DRIVER               <DIR>  01-03-2011  5:15p
FREEDOS              <DIR>  01-03-2011  5:15p
X7SPA                <DIR>  01-03-2011  5:20p
AUTOEXEC BAT           447  01-03-2011  5:15p
COMMAND  COM        66,945  01-03-2011  5:15p
FDCONFIG SYS         1,868  01-03-2011  5:15p
KERNEL   SYS        45,341  01-03-2011  5:15p
         4 file(s)        114,601 bytes
         3 dir(s)       7,124,480 bytes free
A:\>
```
Example BIOS update:
```
A:\X7SPA>cd X7SPA
A:\X7SPA>dir
 Volume in drive A has no label
 Volume Serial Number is 05D8-5287

 Directory of A:\X7SPA

.                    <DIR>  01-03-2011  5:20p
..                   <DIR>  01-03-2011  5:20p
AFUDOS   SMC       150,976  01-03-2011  5:20p
AMI      BAT           104  01-03-2011  5:20p
README~1 TXT         1,266  01-03-2011  5:20p
X7SPA0   C17     4,194,304  01-03-2011  5:20p
         4 file(s)      4,346,650 bytes
         2 dir(s)       7,124,480 bytes free
A:\X7SPA>AMI.BAT X7SPA0.C17
 +---------------------------------------------------------------------------+ 
 |                     AMI Firmware Update Utility  v4.33                    | 
 |      Copyright (C)2009 American Megatrends Inc. All Rights Reserved.      | 
 +---------------------------------------------------------------------------+ 
- Bootblock checksum .... ok
- Module checksums ...... ok
- Warning: No SMBIOS structures are found. Nothing will be preserved in NVRAM.
- Erasing NCB ........... done              
- Writing NCB ........... done              
- Verifying NCB ......... done              
- Erasing flash ......... done              
- Writing flash ......... done              
- Verifying flash ....... done              
- Erasing NVRAM ......... done              
- Writing NVRAM ......... done              
- Verifying NVRAM ....... done              
- Erasing Bootblock ..... done              
- Writing Bootblock ..... done              
- Verifying Bootblock ... done              
- CMOS checksum destroyed
- Program ended normally.

A:\X7SPA>
```

### Appendix: Creating the Initial Image ###

Nfsroot provides a working FreeDOS boot image.  This appendix
documents the procedure used to create that image.  See also the `img`
subdirectory of the nfsroot source code.

Start with `fdboot.img` 1440K floppy image from FreeDOS 1.0 distribution.
Also download `sys-freedos-linux.zip` and `fdfullcd.iso` from the FreeDOS web site.

Unzip `sys-freedos-linux.zip` in the current directory:
```
$ unzip ~/Downloads/sys-freedos-linux.zip
Archive:  /home/garlick/Downloads/sys-freedos-linux.zip
   creating: bootsecs/
  inflating: bootsecs/boot.asm       
  inflating: bootsecs/boot32.asm     
  inflating: bootsecs/copying        
  inflating: bootsecs/boot32lb.asm   
 extracting: bootsecs/version.txt    
  inflating: sys-freedos.pl 
$   
```

Extract `mode.com` out of the `fdfullcd.iso` image:
```
# mount -o loop fdfullcd.iso /mnt
# cp /mnt/freedos/setup/odin/mode.com .
# umount /mnt
#
```

Make sure you have the mtools and nasm packages installed.

Set up `~/.mtoolsrc` with drive x: referring to source image,
and drive y: referring to destination image:
```
drive x: file="fdboot.img"
drive y: file="freedos.img"
```
Create a new FAT file system of about 10 megabytes on y: and copy the contents of `fdboot.img` plus `mode.com` into it:
```
$ mformat -C -t 320 -s 36 -h 2 y:
$ mcopy -s x: y:
$ mcopy mode.com y: y:freedos
$ 
```

Edit the `fdconfig.sys` file to change the `menudefault` and change `command.com` invocation
to source `autoexec.bat`.  The new fdconfig.sys looks like this:
```
; FreeDOS 1.0 Final distro  by Blair Campbell [Blairdude@gmail.com], 
; last update 2005-08-02 by Blair Campbell [Blairdude@gmail.com]
; config.sys loads system drivers. Please edit to suit your needs.

; nfsroot changes
; - make choice 2 the default and only wait 5 seconds before assuming it [jg]

;!SWITCHES=/E
!SWITCHES=/N

menucolor=7,0
MENU     �������������������������������������������������������������������ͻ
MENU     �       FreeDOS 1.0 Final (2006-July-30) INSTALLATION/LIVE CD       �
MENU     �������������������������������������������������������������������͹
MENU     �   1. Install to harddisk using FreeDOS SETUP                      �
MENU     �                                                                   �
MENU     �   2. FreeDOS Safe Mode  (don't load any drivers)                  �
MENU     �                                                                   �
MENU     �   3. FreeDOS Live CD with HIMEM + EMM386                          �
MENU     �                                                                   �
MENU     �   4. FreeDOS Live CD with HIMEM only (default)                    �
MENU     �                                                                   �
MENU     �   5. FreeDOS Live CD only                                         �
MENU     �                                                                   �
MENU     �      FreeDOS is a trademark of Jim Hall 1994-2006                 �
MENU     �������������������������������������������������������������������ͼ
MENUDEFAULT=4,5

134?!DEVICE=A:\DRIVER\HIMEM.EXE
3?!DEVICE=A:\DRIVER\EMM386.EXE X=TEST
12345?!SHELL=A:\COMMAND.COM A:\ /E:2048 /F /MSG /P=A:\AUTOEXEC.BAT
; 34?!DEVICEHIGH=A:\DRIVER\XDMA.SYS
; 345?!DEVICEHIGH=A:\DRIVER\XCDROM.SYS /D:FDCD0000
!DOSDATA=UMB
!DOS=HIGH,UMB
!FILES=20
!BUFFERS=20
!LASTDRIVE=Z
```
The new `autoexec.bat` looks like this
```
REM borrowed from the top of freedos\fdauto.bat
@echo off
SET DEBUG=N
SET NLSPATH=A:\FREEDOS
set dircmd=/P /OGN /4
set lang=EN
SET PATH=A:\FREEDOS;A:\DRIVER

REM get boot arguments
getargs >temp.bat
call temp.bat
del temp.bat

REM serial console redirect
if "%sercons%"=="" goto end
echo Redirecting console to %sercons%
if not "%baudhard%"=="" mode %sercons% baudhard=%baudhard%
if not "%baud%"=="" mode %sercons% baud=%baud%
ctty %sercons%
:end
```
Copy modified `fdconfig.sys` and `autoexec.bat` into the image using mcopy as above.

Sys the new image to make it bootable:
```
$ ./sys-freedos.pl --disk freedos.img --sectors 36 --heads 2

DOS boot sector for freedos.img will be created by:
	nasm -o /dev/stdout -dISFAT16  ./bootsecs/boot.asm
Using FAT16. Partn offset 0, CHS *x2x36  Drive 0, (0x0, 0x29),
SerNo 22A6-C611, Strings 'NO NAME    ',  'FAT16   '.
CHS sector count changed to 36
CHS head count changed to 2
Boot sector successfully updated.
$
```