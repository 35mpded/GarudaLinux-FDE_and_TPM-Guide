### Full Disk Encryption on Garuda Linux backed by TPM 2.0

**Very important note**:
*Make a backup of your data. Any mistakes, while following the guide or incompatabilities may lead to irrevertible data loss!*

# Introduction

Tested on:
* garuda-dr460nized-linux-zen-210406 (5.11.11-zen1-1-zen" and "5.10.28-1-lts) using GUI installer with default partitioning + FDE option.

Requirements:
* System provisioned with TPM
* TPM chip enabled in UEFI settings

*Note: It should work on Arch Linux with minor changes but I haven't tested it.*

## Preparations
***Very important note: Do not reboot your system until you've finished all the steps or you won't be able to boot.***
1. Edit the file /etc/crypttab and change:
	 Choose  A. or B. depending on your partition setup.
	- A. if you are not using swap
	- B. if you using swap

**A (no swap):**

From:
```sh
# <name>               <device>                         <password> <options>
luks-<id> UUID=<id>     /crypto_keyfile.bin luks
```
To:
```sh
# <name>               <device>                         <password> <options>
#luks-<id> UUID=<id>    /crypto_keyfile.bin luks
luks-<id> UUID=<id> none discard
```

**B (swap)**

From:
```sh
# <name>               <device>                         <password> <options>
luks-<id> UUID=<id>     /crypto_keyfile.bin luks
```
To:
```sh
# <name>               <device>                         <password> <options>
#luks-<id> UUID=<id>     /crypto_keyfile.bin luks
luks-<id> UUID=<id>     /crypto_keyfile.bin luks
luks-<id> UUID=<id> none discard
```

2. Delete the file "/crypto_keyfile.bin" (skip this step if you are using swap):
4. Edit the intial ramdisk conf file `/etc/mkinitcpio.conf`
Change this line from (do NOT skip this step, even if you are using swap):
`FILES="/crypto_keyfile.bin"`
to:
`#FILES="/crypto_keyfile.bin"`
6. Based on which method you choose set the hooks in `/etc/mkinitcpio.conf`
Change this line `HOOKS="base udev autodetect modconf block keyboard keymap consolefont plymouth encrypt filesystems"` to the following:

**For Method 1 - Clevis:**
`HOOKS="base udev autodetect modconf block keyboard keymap consolefont clevis encrypt filesystems"`

**For Method 2 - Custom (WIP Avoid for now)**
`HOOKS="base udev autodetect modconf block keyboard keymap consolefont encrypt-tpm encrypt filesystems"`

## Method 1 - Clevis

**Setup**:

1. Install the following packages.
    ```sh
    pacman --needed -S clevis tpm2-tools luksmeta libpwquality
    ```
2. Add `clevis` binding to your LUKS device
note: set the PCR IDs based on your paranoia settings...
    ```sh
    clevis luks bind -d <device> tpm2 '{"pcr_ids":"0,2,4,7"}'
    ```
3. Install the `clevis` hook from [mkinitcpio-clevis-hook](https://github.com/kishorv06/arch-mkinitcpio-clevis-hook)
    ```sh
    ./install.sh
    ```

4. Generate `initramfs` image.
    ```sh
    mkinitcpio -P
    ```
5. Reboot your system.

Now your disk should get decrypted using the key from TPM.

**Note:**
If integrity on your system is changed you will get prompted to manually enter the password for decryption since TPM will not be able to unseal the key.

It is actually recomennded to test this.
1. Open your UFEI settings 
2. Find the TPM settings (most common location is in security)
3. Delete the keys.
4. Boot
Now you will be notified that the TPM key could not be unsealed, and you will be prompted to enter a password for decryption, to fix this follow the next section **"Clevis Binding"**.

**Regenerate Clevis Binding**
To generate a new Clevis pin after changes in system configuration that result in different PCR values, run**

1. Find the slot used for the Clevis pin
`cryptsetup luksDump /dev/sdX`
2. Remove the Clevis binding, run:
`clevis luks unbind -d /dev/sdX -s keyslot`
3. Add a new Clevis binding.
`
clevis luks bind -d <device> tpm2 '{"pcr_ids":"0,2,4,7"}'
`
4. Reboot, the disk now should be decrypted using the key from TPM.

**Avoid password prompt in GRUB (OPTIONAL)**
1.Rebuilt the EFI image using:
```
objcopy \
    --add-section .osrel="/usr/lib/os-release" --change-section-vma .osrel=0x20000 \
    --add-section .cmdline="/proc/cmdline" --change-section-vma .cmdline=0x30000 \
    --add-section .linux="/boot/vmlinuz-linux-zen" --change-section-vma .linux=0x40000 \
    --add-section .initrd="/boot/initramfs-linux-zen.img" --change-section-vma .initrd=0x3000000 \
    "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/boot/efi/EFI/Garuda/grubx64.efi"
```
2. Regenerate the Clevis binding.

Note:
If you are using lts change `vmlinuz-linux-zen.img` and `initramfs-linux-zen.img` to `vmlinuz-linux-lts.img` and `initramfs-linux-lts.img`


## Method 2 - Custom (WIP)

1. Create the key
```
dd if=/dev/random of=/root/secret.bin bs=32 count=1
```

2. Add the key to luks (if you are using SWAP do it for both encrytped drives)
```
cryptsetup luksAddKey /dev/<your drive> /root/secret.bin
```

3. Add key to TPM
```
tpm2_createpolicy --policy-pcr -l sha1:0,2,4,7 -L policy.digest
tpm2_createprimary -C e -g sha1 -G rsa -c primary.context
tpm2_create -g sha256 -u obj.pub -r obj.priv -C primary.context -L policy.digest -a "noda|adminwithpolicy|fixedparent|fixedtpm" -i /root/secret.bin
tpm2_load -C primary.context -u obj.pub -r obj.priv -c load.context
tpm2_evictcontrol -C o -c load.context 0x81000000
rm load.context obj.priv obj.pub policy.digest primary.context
```

4. Unseal:
```sh
tpm2_unseal -c 0x81000000 -p pcr:sha1:0,7 -o /crypto_keyfile.bin
```

5. Rebuild inital ramdisk 
```
mkinitcpio -P
```
6. Avoid password prompt in GRUB (OPTIONAL)
Rebuilt the EFI image using:
```
objcopy \
    --add-section .osrel="/usr/lib/os-release" --change-section-vma .osrel=0x20000 \
    --add-section .cmdline="/proc/cmdline" --change-section-vma .cmdline=0x30000 \
    --add-section .linux="/boot/vmlinuz-linux-zen" --change-section-vma .linux=0x40000 \
    --add-section .initrd="/boot/initramfs-linux-zen.img" --change-section-vma .initrd=0x3000000 \
    "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/boot/efi/EFI/Garuda/grubx64.efi"
```
After reboot you should get prompted to input the password manually. This behaviour is expected since you change your EFI image.
Remove Remove key from TPM and redo step 3. :
```sh
tpm2_evictcontrol -C o -c 0x81000000
```

...

## Some other important notes:
If the system un-expectedly asks for LUKS password after reboot it may indicate that your system was compromised.

Always test your system to see if the TPM is handled properly. 
I suggest the following procedure:
1. Bind the TPM keys, test if you sucesfully boot without issues.
   a. If you dont boot: troubleshoot .
   b. If you do boot: continue to next step.
2. Clear/delete the keys stored on TPM, preferbly by using the UEFI menu.
   a. If the system boots: you have a serious issue and you should trobleshoot.
   b. If the system prompts for password input: you are good to go, just regenerate the TPM binding.


Sources:

https://wiki.archlinux.org/index.php/Trusted_Platform_Module

https://github.com/pawitp/arch-luks-tpm

https://pawitp.medium.com/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704

https://github.com/kishorv06/arch-mkinitcpio-clevis-hook

https://bentley.link/secureboot/

https://github.com/archont00/arch-linux-luks-tpm-boot

https://github.com/saucepan14/TPMSecuredArch
