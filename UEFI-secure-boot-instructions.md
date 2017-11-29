# Boot FreeBSD With UEFI Secure Boot

***

## Contents:

- Introduction
- 1. Building The Loader With Memory Disk Support
- 2. Signing An EFI Binary
- 3. Enrolling Secure Boot Keys In Firmware
- Additional Resources

## Introduction:
Secure boot provides a way to ensure that only authorized EFI binaries are loaded by a computer's firmware. This ensures that no malicious code can be run before the operating system is loaded. This document describes the steps to secure FreeBSD's boot process. FreeBSD's regular EFI boot process has two stages: `boot1.efi` and `loader.efi`. The boot1 binary is loaded by the EFI firmware. The loader binary is then loaded by boot1, after which it loads the kernel. Ideally each step of this process would involve a cryptographic handshake; boot1 would verify the loader which would in turn verify the kernel, thereby ensuring that only authorized code is run. Currently there is no support for these verifications. However, the goal of securely booting the kernel can still be achieved; to do so it is necessary to combine the loader and kernel into a single object which can then be securely loaded by the EFI firmware. There are three steps in this process: building the composite loader/kernel object, signing the composite object, authorizing the composite object in the EFI firmware. These steps are laid out below.

|Note:|
|:---|
|These instructions assume you are following them from an existing FreeBSD installation on an EFI enabled computer with secure boot turned off in the  firmware menu. If this is not the case the steps below will need to be adjusted appropriately.|


## 1. Building The Loader With Memory Disk Support
To build a bootable loader/kernel object it is necessary to reserve memory in  the loader binary for a boot filesystem where the kernel will be stored. Before  memory can be reserved in the loader binary the boot filesystem image must be created so that we know how much memory to reserve. The boot filesystem image outlined below includes:

1. the kernel
2. kernel object directory
3. the loader FORTH scripts


	$ mkdir ~/bootfs
	$ mkdir ~/bootfs/boot
	$ mkdir ~/bootfs/etc
	$ cd ~/bootfs/boot
	$ cp -r /boot/kernel .
	$ cp /boot/*.4th .
	$ cp -r /boot/defaults .
	$ cp /boot/loader.conf .
	$ cp /boot/*.rc .
	$ cp /boot/device.hints .
	$ cp /boot/loader.help .


During boot the kernel mounts the filesystems found in `/etc/fstab` on the device the loader booted from. Since the kernel will be booting from a memory disk we must include an appropriate `/etc/fstab` in order for the kernel to mount the required filesystems.

	$ cp /etc/fstab ~/bootfs/etc/fstab

create the filesystem image:

	$ cd ~/
	$ makefs bootfs.img bootfs

determine the size of the boot filesystem. This will determine how much memory to reserve in the loader:

	$ ls -l bootfs.img | awk '{print $5}'

add a safety factor of a couple hundred bytes (~512) to this number, and record it. Proceed to build the loader:

	$ cd /usr/src/sys/boot
	$ make MD_IMAGE_SIZE=${BOOTFS_SIZE_PLUS_SAFETY}

|Note:|
|:----|
|To clean after building the loader you must run `make clean`, with `MD_IMAGE_SIZE` defined, otherwise the memory disk object will not be cleaned. i.e:|

	$ make MD_IMAGE_SIZE=0 clean

install the loader binaries:

	$ sudo make install

embed the bootfs image into the loader:

	$ sudo /usr/src/sys/tools/embed_mfs.sh /boot/loader.efi ~/bootfs.img

note you may need to make `embed_mfs.sh` executable:

	$ chmod +x /usr/src/sys/tools/embed_mfs.sh

The loader/bootfs object is now ready to use, however in order to boot from it it must by moved to the EFI system partition (ESP). The ESP is a small vFAT partition (~100MB - ~512MB) with an 'efi' label. This is where the EFI firmware looks for bootloaders. Find the ESP and mount it:

	$ gpart show
	$ mount -t msdosfs /dev/${ESP_DEVICE} /mnt

The exact location to place the loader binary in the ESP is somewhat dependent on the computer's firmware. In principle one should be able to place the loader binary anywhere on the ESP and then direct the firmware to boot from it via the firmware menu (usually accessed by pressing F12, esc or delete during startup). However some EFI firmware implementations are less than standards compliant and so you might have to experiment with different locations/firmware settings. In practice `${ESP_MOUNTPOINT}/efi/boot/bootx64.efi` and `${ESP_MOUNTPOINT}/efi/boot/bootia32` are generally safe bets for x86_64/amd64 and ia32/i386 computers respectively. for example:

|WARNING:|
|--------|
|you may already have a `bootx64.efi` loader in your ESP don't overwrite it unless you know what you're doing.|

	$ sudo cp /boot/loader.efi /mnt/efi/boot/
	$ cd /mnt/efi/boot
	$ sudo mv loader.efi bootx64.efi

At this point it is a good idea to test that the composite loader will boot the system. turn off secure boot in the firmware and boot the computer. Depending on the size of yout bootfs image there may be a noticeable delay before the loader menu appears. At the loader menu hit `Esc`. An example loader prompt session confirming that the memory disk is selected as the boot device and that it contains the boot filesystem:

	OK show currdev
	md0
	OK ls
	/
	 d boot
	 d etc

once you're satisfied that the correct device has been selected run:

	OK boot

to finish booting the system. If everything went well the kernel should boot and mount `/` successfully.

## 2. Signing An EFI Binary
Now that we have a working loader/bootfs EFI binary it is necessary to sign it so that the computer's firmware will recognize it an authorized boot loader. save the following script as `mkkey.sh` and make sure it is executable:

	mkkey.sh:
---
	#!/bin/sh
	/usr/share/examples/uefisign/uefikeys PK
	/usr/share/examples/uefisign/uefikeys KEK
	/usr/share/examples/uefisign/uefikeys DB
	
	GUID=`python -c 'import uuid; print(str(uuid.uuid1()))'`
	echo $GUID > myGUID.txt
	
	cert-to-efi-sig-list -g $GUID PK.pem PK.esl
	cert-to-efi-sig-list -g $GUID KEK.pem KEK.esl
	cert-to-efi-sig-list -g $GUID DB.pem DB.esl
	rm -f noPK.esl
	touch noPK.esl
	
	sign-efi-sig-list -t "$(date +'%Y-%m-%d %H:%M:%S')" \
		-k PK.key -c PK.pem PK PK.esl PK.auth
	sign-efi-sig-list -t "$(date +'%Y-%m-%d %H:%M:%S')" \
		-k PK.key -c PK.pem PK noPK.esl noPK.auth
	sign-efi-sig-list -t "$(date +'%Y-%m-%d %H:%M:%S')" \
		-k PK.key -c PK.pem KEK KEK.esl KEK.auth
	sign-efi-sig-list -t "$(date +'%Y-%m-%d %H:%M:%S')" \
		-k KEK.key -c KEK.pem db DB.esl DB.auth
	
	chmod 0600 *.key

|Note:|
|:----|
|`mkkey.sh` was adapted from the mkkey.sh script found here: <http://www.rodsbooks.com/efi-bootloaders/controlling-sb.html>|

generate the keys and certificates:

	./mkkey.sh

`mkkey.sh` produces `*.pem` certificates, `*.key` private keys, `*.esl` EFI  signature lists, `*.auth` signed EFI signature lists and `*.cer` certificates. In step 3 the `*.esl`, `*.auth` or  `*.cer` certificates will be enrolled in the EFI firmware. The files `PK.key`, `KEK.key` and `DB.key` are private keys. Keep them safe. Sign the loader/bootfs binary located on the ESP. From the directory the keys were generated in:

	sudo uefisign -c DB.pem -k DB.key \
	-o /mnt/efi/boot/bootx64-signed.efi /mnt/efi/boot/bootx64.efi

To ensure that you will automatically boot from the signed binary you may have
to change its name to `bootx64.efi` (see previous section). 

## 3. Enrolling Secure Boot Keys In Firmware
In order for the firmware to recognize signed binaries and reject unsigned ones it is necessary to enroll certificates in the EFI firmware. There are three  keys that are standard to every UEFI secure boot system. The platform key (PK), the key exchange key (KEK), and the database key (DB). The PK is the top level of security. Once the PK has been enrolled no other keys can be enrolled or  modified unless they are signed by the PK. The PK is not used for signing  binaries and there can only be one PK enrolled at a time. The KEK is the next level of security. It is used to authorize the exchange of DB keys, and may be used to sign binaries. There can be multiple KEKs enrolled at the same time. The DB keys are used to sign binaries and are secured by the KEK. In order to boot the signed composite loader/bootfs object all three keys must be enrolled.

**UEFI secure boot modes:**

There are three secure boot modes: 'setup mode/secure boot off', 'user mode/secure boot off', and 'user mode/secure boot on'. In setup mode an EFI binary may be run and new keys may be enrolled. In user mode with secure boot off any EFI binary may be run, and only signed keys can be enrolled or modified. In user mode with secure boot on only signed binaries may be run and only signed keys may be enrolled.

**Certificate file types:**

Depending on the tool used to enroll keys in the firmware either `*.cer`, `*.esl` or `*.auth` files will be used. `*.cer` and `*.esl` files are unsigned certificates and can only be enrolled when the firmware is in setup mode. The `*.auth` files generated by `mkkey.sh` are `*.esl` files that have been signed by the PK. this means that they can be enrolled even if the firmware is in user mode/secure bot on with the PK enrolled. Depending on the firmware implementation the PK might have to be a `*.auth` file that has been signed by itself. `mkkey.sh` generates a self signed platform key (`PK.auth`). Additionally `mkkey.sh` generates a null PK (`noPK.auth`) which is signed by `PK.auth`, enrolling `noPK.auth` allows one to enroll arbitrary keys without switching firmware modes.

**Tools to enroll keys:** 

If you're lucky the firmware on your target computer will support key enrollment. If this is not the case then the 'KeyTool' EFI binary provided by the Linux® `efitools` package is a good option. Begin by downloading and extracting `efitools`. Copy `KeyTool.efi` to the ESP. Reboot the computer and enter the firmware menu. Regardless of which tool is used to enroll keys it is necessary to place the computer in setup mode. The exact steps to do this are heavily dependent on the firmware. If the firmware shipped with keys already enrolled they may be destroyed when setup mode is set. If you need to back up  these manufacturer keys you can use KeyTool's 'Save Keys' option. 

|WARNING:|
|--------|
|Destroying or removing manufacturer secure boot keys can be dangerous in a dual boot environment, as it may prevent the other OS's boot loader from running. Additionally removing manufacturer keys may make it difficult to update the computer's firmware in the future.|

**Using KeyTool:**

The EFI firmware should provide an option to choose which file to boot from. In general it will provide a file discovery interface that will allow you to browse the ESP and select a `.efi` binary to execute. Use this system to launch `KeyTool.efi`. 

KeyTool provides a full screen text based menu that allows you to modify the keys present on the system. At the top of the screen KeyTool lists the mode that the firmware is currently in. The cursor keys are used to navigate and the `Enter` key selects an option. `Esc` returns to the previous menu. KeyTool only allows one to browse the ESP so the certificate to be enrolled must be present on the ESP. Additionally KeyTool only shows `*.esl` and `*.auth` files. When using KeyTool it is necessary to enroll keys in reverse order, i.e. DB must be enrolled before KEK, which must be enrolled before PK, unless keys signed by the PK are available. This is because security changes are immediate, once the PK has been enrolled only keys signed by it may be enrolled after it. 

Once keys have been successfully enrolled the final step is to place the firmware in user mode and turn secure boot on. Note once secure boot is enabled KeyTool will no longer have authorization to run since it is not signed. If you wish to run KeyTool after turning on secure boot it is necessary to sign the binary with `uefisign(8)`.

## Additional Resources:
Information about how to setup EFI boot under Linux is available here: 

<http://www.rodsbooks.com/efi-bootloaders/index.html>

The majority of information in this document has been adapted from the above 
source for use with FreeBSD. Additionally the `mkkey.sh` script found in step 2 
was adapted from the script found on the above site.

the Debian Linux® `efitools` package can be downloaded from here:

<https://packages.debian.org/buster/amd64/efitools/download>

At the time of this writing Debian's efitools is only available from the testing branch and only supports x86_64/amd64 architecture. If the above link is broken a search for 'efitools' should reveal alternative sources for the package. Debian's `.deb` package format can be extracted on FreeBSD as follows:

	$ ar vx ${PACKAGE_NAME}.deb
	$ tar -xzvf data.tar.gz

this will produce a directory structure containing the `KeyTool.efi` binary.


**Authors:**

Zakary Nafziger (zsnafzig_at_edu.uwaterloo.com)
 
Written 27 October 2017