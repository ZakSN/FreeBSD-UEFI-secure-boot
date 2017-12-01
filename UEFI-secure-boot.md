# FreeBSD UEFI Secure Boot

***

## Contents
1. Introduction
2. Build The Loader With Embedded Memory Disk Support
	1. Create Boot Image
	2. Build Loader With Extra Space
	3. Embed Boot Image In Loader
3. Signing An EFI binary
4. Securing A Platform
	1. Qemu
		1. Obtain Secure Boot Enabled UEFI Firmware
		2. Enroll Keys
		3. Boot
		4. Notes About Secure Boot in Qemu
	2. Real Hardware
		1. Enroll Keys From Firmware
		2. Enroll Keys With KeyTool
		3. Boot
5. Additional Resources


## 1. Introduction
Secure boot provides a way to ensure that only authorized EFI binaries are loaded by a computer's firmware. This ensures that no malicious code can run before the operating system is loaded. This document describes one method of securing FreeBSD's boot process. FreeBSD's regular UEFI boot process has two stages: `boot1.efi` and `loader.efi`. The `boot1.efi` binary is loaded by the UEFI firmware. The `loader.efi` binary is then loaded by `boot1.efi`, after which it loads the kernel. Ideally each step of this process would involve a cryptographic handshake; `boot1.efi` would verify `loader.efi` which would in turn verify the kernel, thereby ensuring that only authorized code is run. Currently there is no support for these verifications. However, the goal of securely booting the kernel can still be achieved. To do so it is necessary to combine the loader and kernel into a single object which can then be securely loaded by the UEFI firmware. This is the method described in this document.

Further Information about FreeBSD's UEFI boot process can be found here: <https://wiki.freebsd.org/UEFI>. Other possible strategies for securely booting FreeBSD are outlined here: <https://wiki.freebsd.org/SecureBoot>.


## 2. Build The Loader With Embedded Memory Disk Support
To build a bootable loader/kernel object it is necessary to reserve memory in  the `loader.efi` binary for a boot filesystem image where the kernel will be stored. Before  memory can be reserved in the `loader.efi` binary the boot filesystem image must be created so that the correct amount of space can be reserved in the loader. An appropriately sized `loader.efi` can then be built and the boot filesystem can be embedded in it.


### 2.1. Create Boot Image
The boot filesystem outlined below contains: the kernel, the kernel object directory, and the loader FOURTH scripts.

|Note|
|----|
|These instructions produce a boot filesystem identical to the one on the computer they are being run on. If the target platform needs a different boot configuration the below instructions may need to be adjusted|

Create a directory with the required boot files:
```
$ mkdir ~/bootfs
$ mkdir ~/bootfs/boot
$ cd ~/bootfs/boot
$ cp -r /boot/kernel .
$ cp /boot/*.4th .
$ cp -r /boot/defaults .
$ cp /boot/loader.conf .
$ cp /boot/*.rc .
$ cp /boot/device.hints .
$ cp /boot/loader.help .
```

During boot the kernel mounts the filesystems found in `/etc/fstab` on the device the loader booted from. Since the kernel will be booting from a memory disk we must include an appropriate `/etc/fstab` in order for the kernel to mount the required filesystems.
```
$ mkdir ~/bootfs/etc
$ cp /etc/fstab ~/bootfs/etc/fstab
```

create the filesystem image:
```
$ cd ~/
$ makefs bootfs.img bootfs
```

determine the size of the boot filesystem. This will determine how much memory to reserve in the loader:
```
$ ls -l bootfs.img | awk '{print $5}'
```
add a safety factor of a couple hundred bytes (~512) to this number, and record it. 


### 2.2. Build the Loader With Extra Space
```
$ cd /usr/src/stand
$ make MD_IMAGE_SIZE=${BOOTFS_SIZE_PLUS_SAFETY}
```

|Note|
|-----|
|To clean after building the loader you must run `make clean`, with `MD_IMAGE_SIZE` defined, otherwise the memory disk object will not be cleaned. i.e:|
|`$ make MD_IMAGE_SIZE=0 clean`|


### 2.3. Embed Boot Image In Loader
To embed `bootfs.img` into the loader:
```
$ cp /usr/obj/${PATH_TO_LOADER}/loader.efi ~/
$ /usr/src/sys/tools/embed_mfs.sh ~/loader.efi ~/bootfs.img
```

note you may need to make `embed_mfs.sh` executable:
```
$ chmod +x /usr/src/sys/tools/embed_mfs.sh
```


## 3. Signing An EFI binary
FreeBSD's base system includes a tool for signing EFI executables: uefisign(8). Additionally the script `/usr/share/examples/uefisign/uefikeys` is provided to generate the keys and certificates needed to sign an EFI executable. The general procedure for generating keys and certificates and using them to sign an executable is found in the `uefisign` manual page, quoted below:

>Generate self-signed certificate and use it to sign a binary:
>```
>           /usr/share/examples/uefisign/uefikeys testcert
>           uefisign -c testcert.pem -k testcert.key -o signed-binary binary
>```

`uefikeys` produces three types of files: `*.pem`, `*.cer`, and `*.key` each named after the argument passed to `uefikeys`. The `*.pem` and `*.cer` files are the ascii and binary encoding of the certificate that will be used to sign the EFI executable.

|Note|
|----|
| Most UEFI documentation refers to the public certificates as 'keys' or 'public keys' rather than certificates. This convention is followed in this document.|

`uefikeys` is a shell script wrapper for openssl that generates self-signed certificates. UEFI firmware implementations are supposed to trace keys back to the root of trust (generally a self-signed platform key), however they should also accept self-signed keys. The trust hierarchy between UEFI keys can get complicated especially on dual boot systems. For the purposes of this document only simple self-signed keys will be dealt with. More information about how UEFI keys work: <http://www.rodsbooks.com/efi-bootloaders/controlling-sb.html>.


## 4. Securing A Platform
After creating a signed bootable loader/kernel object, it is necessary to enable secure boot and enroll keys in the computer's firmware. This section includes information about setting up and securing a virtual environment in Qemu as well as some information about how to secure actual hardware


### 4.1. Qemu
Qemu can run UEFI secure boot firmware if provided with a suitable firmware binary. Finding suitable firmware proved to be somewhat difficult.

 
#### 4.1.1. Obtain Secure Boot Enabled UEFI Firmware
An open source reference implementation of UEFI firmware is provided by the TianoCore EDK II project. The Open Virtual Machine Firmware (OVMF) produced by TianoCore is UEFI firmware that is compatible with Qemu, however no OVMF ports are available for FreeBSD and the binary OVMF packages distributed by the most major Linux distributions don't seem to include secure boot support. The good news is that since OVMF is firmware it is operating system agnostic so it doesn't matter where the OVMF binary comes from; it should still work with Qemu on FreeBSD.

It is hypothetically possible to build an OVMF binary with the EDK II development kit, however doing so on FreeBSD might require a significant amount of effort. TianoCore's build instructions can be found here:  <https://github.com/tianocore/tianocore.github.io/wiki/Getting-Started-with-EDK-II> and the instructions for building OVMF are here: <https://github.com/tianocore/tianocore.github.io/wiki/OVMF>.

The only place that the author has successfully sourced a secure boot enabled OVMF binary is Arch Linux's Arch User Repository (AUR). There may be easier ways to procure an OVMF binary with secure boot enabled, but the following method does work as of this writing.

The AUR package that provides OVMF with secure boot is `ovmf-git` (<https://aur.archlinux.org/packages/ovmf-git/>). Unfortunately the software is distributed as source and it uses the Arch Build System (similar idea to Ports) on the target computer to get built. The upshot is that a computer running Arch Linux is required in order to produce a working OVMF binary. Additionally the `ovmf-git` package is currently somewhat broken; it does not build the IA32 version of OVMF. The workaround is to modify the build script to avoid building the IA32 binary and only build the x86_64 binary. In general the process the author used to build the OVMF binary was as follows:

1. get a computer (or emulator) that is running Arch
2. follow one's prefered steps for installing something from the AUR
3. modify the `ovmf-git` PKGBUILD build script as per user Asura's pinned comment: <https://aur.archlinux.org/packages/ovmf-git/?comments=all> (can't link to specific comment, should be the second from the top unless this has been fixed by the time this is being read)
4. proceed with the build (actual build was pretty fast)
5. copy the `ovmf_x64.bin` firmware binary to the computer running Qemu


#### 4.1.2. Enroll Keys
To use secure boot enabled OVMF with `qemu-system-x86_64` a few command line options are required:
- `-m 1024` ensure that the VM has enough memory
- `-bios ${PATH_TO_OVMF_BINARY}` direct qemu to use the OVMF binary procured in section 4.1.1
- Qemu is able to emulate a virtual FAT filesystem. This particularly useful for testing with UEFI since the EFI system partition (ESP) must be FAT formatted: `-hda fat:${PATH_TO_EFI_DIRECTORY}`
- alternatively you can create an emulated hard drive and go though the usual steps required to create a FAT formatted EFI system partition.

regardless of how the ESP is created it will need to provisioned with at least the following files:
- The key (*.cer) generated in section 3
- the signed EFI binary produced in section 3
- Optionally: an unsigned EFI binary to test with. It's a good idea to keep an unsigned EFI binary on the ESP to ensure that the firmware is actually rejecting unsigned software when secure boot is enabled.

It should now be possible to start Qemu with secure boot enabled. Unlike some UEFI firmware that ships with consumer products OVMF provides an interface for enrolling secure boot keys, without the need to use a third party tool. The menu path: `Device Manager` -> `Secure Boot Configuration` and select `Custom Mode` in the `Secure Boot Mode` option. Then select `Custom Secure Boot Options`. This should present a series of options for each type of secure boot key. The bare minimum needed for secure boot to reject unsigned EFI binaries and boot signed EFI binaries is a self signed Platform Key (PK) and a self signed Database Key (DB). The PK enables secure boot and the Database key is used to sign EFI applications. For the purposes of this document the PK and DB can be the same self signed certificate. For more complex configurations it may be necessary to have keys signed by other keys, this is common when dual booting two OSes (more information in section 5 reference [3]). To enroll the key generated in section 3 as the PK select `PK Options` -> `Enroll PK` -> `Enroll PK Using File` -> select the partition containing the FAT filesystem from the available options. The partitions are labelled awkwardly, but most of the time there is only one FAT partition available. -> browse to the `*.cer` file generated in section 3 and select it -> `Commit Changes and Exit`. The process is similar for enrolling the DB key.


#### 4.1.3. Boot
After changing the PK it is necessary to reset the VM. This will be done automatically after exiting the firmware menu (by launching the internal shell for example). Alternatively the VM can be reset via the `reset` option on the top level menu. After the reset the VM should be in secure boot mode. 

Testing secure boot functionality can be done from the internal shell. The internal shell can be accessed from the top level menu via `Boot Manager` -> `EFI Internal Shell`. Once at the `Shell>` prompt is necessary to select the FAT file system with the test executable(s). example:
```
Shell> FS0:
```
simple MSDOS-ish or UNIX-ish shell commands will work from here. Navigate to the EFI executable and run it.

|Note|
|----|
|Qemu does not emulate NVRAM so no EFI variables are saved between sessions. This means that it is necessary to re-enroll secure boot keys every time the VM is started. EFI vars are saved across resets.|


#### 4.1.4. Notes About Secure Boot in Qemu
`uefikeys` uses the provided argument as the common name of the certificate generated (`openssl`'s `-subj` field). OVMF seems to care about the common name of certificate. Whether or not this is a bug is unclear. However, the behaviour seems to be consistently present in the few versions of OVMF that were tested with. OVMF won't run an EFI binary that has been signed with a certificate that was generated with one of the following strings: "PK", "testcert", however it will run and EFI binary that was signed with one of these strings: "test", "cert". If OVMF inexplicably fails to run an EFI binary with the error: `Command Error Status: Security Violation` it might be worth trying a key pair that was generated with the common name "test". This is exactly as weird as it sounds.


### 4.2. Real Hardware
In order for the firmware to recognize signed binaries and reject unsigned ones it is necessary to enroll keys (certificates, section 3) in the UEFI firmware. There are three  keys that are standard to every UEFI secure boot system. The platform key (PK), the key exchange key (KEK), and the database key (DB). The UEFI keys can be used to establish complex security hierarchies (section 5 [3]), however for the purposes of this document only self-signed keys will be considered. 


#### 4.2.1. Enroll Keys From Firmware
Ideally the Firmware on the target computer would provide a method for modifying the secure boot keys. If this is the case the steps should be roughly similar to those outlined in section 4.1.2, however the exact interface may vary. Hopefully the firmware UI is self explanatory. If the firmware on the target computer does not provide an interface for modifying secure boot keys the EFI application 'KeyTool' may be a good option. 


#### 4.2.2. Enroll Keys With KeyTool
KeyTool is an EFI application that provides a secure boot key enrollment interface. Currently no FreeBSD port provides KeyTool, however it is available from the Linux `efitools` package, which is distributed by a number of major distributions. Begin by downloading and extracting `efitools` (section 5, [6]). Copy `KeyTool.efi` to the ESP. Reboot the computer and enter the firmware menu. It is necessary to place the computer in setup mode (secure boot off). If the firmware shipped with keys already enrolled they may be destroyed when setup mode is set. If you need to back up  these manufacturer keys you can use KeyTool's 'Save Keys' option.

KeyTool is picky about file types and will not allow a self signed `*.cer` file to be enrolled as the PK. KeyTool only seems to support enrolling self-signed `*.auth` files as the PK. `*.auth` files are signed "efi signature list" (`*.esl`) files, `*.esl` and `*.auth` files can be generated from `*.pem` certificates with the `cert-to-efi-sig-list` and `sign-efi-sig-list` utilities respectively. `cert-to-efi-sig-list` and `sign-efi-sig-list` are both provided by the `efitools` package. Section 5 [3] provides a lot more information about how to use KeyTool.

Since the KeyTool binary is unsigned it can not be run after secure boot is turned on meaning that keys cannot be modified while secure boot is on. If it is necessary to run KeyTool without turning off secure boot KeyTool can be signed with uefisign(8) like any other EFI application.


#### 4.2.3. Boot
Once Keys have been enrolled in the firmware and it has been placed in "user mode" (secure boot on) it should boot signed binaries and reject unsigned binaries. It is a good idea to test that everything is working by copying an unsigned EFI binary to the ESP in addition to a signed EFI binary to make sure that the firmware rejects the unsigned binary.


## 5. Additional Resources
[1]. Information on FreeBSD's UEFI boot process: <https://wiki.freebsd.org/UEFI>

[2]. Information about FreeBSD's secure boot support: <https://wiki.freebsd.org/SecureBoot>

[3]. Information about how UEFI keys work: <http://www.rodsbooks.com/efi-bootloaders/controlling-sb.html>. The rest of this site also has lots of information about U/EFI firmware and bootloaders: <http://www.rodsbooks.com/efi-bootloaders/index.html>. The site is targeted at a desktop Linux audience, but a good deal of the information is platform agnostic since it deals with firmware.

[4]. TianoCore's setup and build instructions for EDK II and OVMF: <https://github.com/tianocore/tianocore.github.io/wiki/Getting-Started-with-EDK-II>, <https://github.com/tianocore/tianocore.github.io/wiki/OVMF>

[5]. Arch User Repository page for `ovmf-git`: <https://aur.archlinux.org/packages/ovmf-git/>, PKGBUILD modification instructions <https://aur.archlinux.org/packages/ovmf-git/?comments=all>

[6]. the Debian Linux `efitools` package can be downloaded from here:

<https://packages.debian.org/buster/amd64/efitools/download>

At the time of this writing Debian's efitools is only available from the testing branch and only supports x86_64 architecture. If the above link is broken a search for 'efitools' should reveal alternative sources for the package. Debian's `.deb` package format can be extracted on FreeBSD as follows:

	$ ar vx ${PACKAGE_NAME}.deb
	$ tar -xzvf data.tar.gz

this will produce a directory structure containing the `KeyTool.efi` binary.

***

**Authors:**

Zakary Nafziger (zsnafzig_at_edu.uwaterloo.com)
 
Written 2017-12-1