## Secure boot in Qemu using TianoCore/EDK II OVMF on a FreeBSD host

Date of writing: 2017-11-29

1. Obtain a secure boot enabled OVMF binary
2. Sign an EFI executable
3. Start Qemu and enroll secure boot keys

### 1. Obtain a secure boot enabled OVMF binary
Secure boot enabled OVMF binaries are not available within the ports collection. There are two options for obtaining a firmware binary: get one packaged by a Linux distribution, or build it yourself.

#### 1.1 Obtaining a binary from Linux:
I've had some difficulty finding a secur boot enabled binary from any Linux distribution. Eventually I got a package from Arch Linux's Arch User Repository (AUR) to work. There may be an easier method of obtaining the necessary binary. The AUR package I ended up using was: <https://aur.archlinux.org/packages/ovmf-git/>. The package does not properly build the IA32 OVMF binary so it is necessary to edit the PKGBUILD so that it only builds the x86_64 firmware. Instructions  for the necessary modifications can be found in the comments of the AUR webpage (linked above).

#### 1.2 Building a binary:
The TianoCore wiki has instructions for setting up the EDK II environment and using it to build firmware, setup instructions: <https://github.com/tianocore/tianocore.github.io/wiki/Getting-Started-with-EDK-II> instructions for building OVMF: <https://github.com/tianocore/tianocore.github.io/wiki/OVMF>

### 2 Sign an EFI executable
FreeBSD's base system includes a tool for signing EFI executables: `uefisign`(8). Additionally the script `/usr/share/examples/uefisign/uefikeys` is provided to generate the keys and certificates needed to sign an EFI executable. The general procedure for generating keys and certificates and using them to sign an executable is found in the `uefisign` manual page, quoted below:

Generate self-signed certificate and use it to sign a binary:
```
           /usr/share/examples/uefisign/uefikeys testcert
           uefisign -c testcert.pem -k testcert.key -o signed-binary binary
```

`uefikeys` produces three types of files: `*.pem`, `*.cer`, and `*.key` each named after the argument passed to `uefikeys`. The `*.pem` and `*.cer` files are the ascii and binary encodings of the certificate that will be used to sign the efi executable. OVMF only understands binary encoded `.cer` certificates although other firmware implementations may require different formats. The `.key` file is the private key.

| Note|
|-----|
|`uefikeys` uses the provided argument as the common name of the certificate generated (`openssl`'s `-subj` field). OVMF seems to care about the common name of certificate. I'm not sure if this is a bug or not but the behaviour seems to be consistently present in the few versions of OVMF I've tested with. OVMF won't run an EFI executable that has been signed with a certificate that was generated with one of the following strings: "PK", "testcert", however it will run and EFI executable that was signed with one of these strings: "test", "cert". If OVMF inexplicably fails to run an EFI executable with the error: `Command Error Status: Security Violation` you might want to try using a key pair that was generated with the common name "test". This is exactly as weird as it sounds.|

### 3 Start Qemu and enroll secure boot keys
To use secure boot enabled OVMF with `qemu-system-x86_64` a few command line options are required:
- `-m 1024` ensure that the VM has enough memory
- `-bios ${PATH_TO_OVMF_BINARY}` direct qemu to use the OVMF binary procured in step 1
- Qemu is able to emulate a virtual FAT filesystem. This particularly useful for testing with UEFI since the EFI system partition (ESP) must be FAT formatted: `-hda fat:${PATH_TO_EFI_DIRECTORY}`
- alternatively you can create an emulated hard drive and go though the usual steps required to create a FAT formatted EFI system partition.

regardless of how the ESP is created it will need to provisioned with at least the following files:
- The certificate generated in the previous step
- A signed EFI executable to test with
- Optionally: an unsigned EFI executable to test with. It's a good idea to keep an unsigned EFI executable on the ESP so that you can make sure that the firmware is rejecting unsigned software when secure boot is enabled.

|Note|
|----|
|In most UEFI documentation the public certificate used to sign something is referred to as a key. This convention will be used below, so keep in mind that "key" means "certificate" from here on.|

You should now be able to start a Qemu VM with secure boot enabled. Unlike some UEFI firmware that ships with consumer products OVMF provides an interface for enrolling secure boot keys, without the need to use a third party tool. The menu path: `Device Manager` -> `Secure Boot Configuration` and select `Custom Mode` in the `Secure Boot Mode` option. Then select `Custom Secure Boot Options`. This should present a series of options for each type of secure boot key. The bare minimum needed for secure boot to reject unsigned EFI executables and boot signed EFI executables is a self signed Platform Key (PK) and a self signed Database Key (DB). The PK enables secure boot and the Database key is used to sign EFI applications. For our purposes the PK and DB can be the same self signed certificate. For more complex configurations it may be necessary to have keys signed by other keys, this is common when dual booting two OSes. To enroll the key generated in part 2 as the PK select `PK Options` -> `Enroll PK` -> `Enroll PK Using File` -> select the partition containing the FAT filesystem from the available options. The partitions are labelled awkwardly, but most of the time there is only one FAT partition available. -> browse to the `*.cer` file generated in step 2 and select it -> `Commit Changes and Exit`. The process is similar for enrolling the DB key.

After changing the PK it is necessary to reset the VM. This will be done automatically after exiting the firmware menu (by launching the internal shell for example). Alternatively the VM can be reset via the `reset` option on the top level menu. After the reset the VM should be in secure boot mode. 

Testing secure boot functionality can be done from the internal shell. The internal shell can be accessed from the top level menu via `Boot Manager` -> `EFI Internal Shell`. Once at the `Shell>` prompt is necessary to select the FAT file system with the test executable(s). example:
```
Shell> FS0:
```
simple MSDOS-ish or UNIX-ish shell commands will work from here. Navigate to the EFI executable to run.

|Note|
|----|
|Qemu does not emulate NVRAM so no EFI variables are saved between sessions. This means that it is necessary to re-enroll secure boot keys every time the VM is started. EFI vars are saved across resets.|

**Authors:**

Zakary Nafziger (zsnafzig_at_edu.uwaterloo.com)