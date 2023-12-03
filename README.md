# mkfeud
Bash script to create SecureBoot-able, fully encrypted (boot and root partitions), Ubuntu USB sticks.

See the header in the mkfeud file for more information on how to run mkfeud. Alternatively, run:
```sh
mkfeud --help
```

## Why?

An extremely simplified view of the boot process for Ubuntu is presented here. The purpose of this short explanation is to provide context on why SecureBoot and full disk encryption are important, as well as potential shortcomings. **Please note that I am by no means an expert, what is described below may be imprecise or just plain wrong.**

TLDR:
1. UEFI verifies shim's signature and runs it
2. shim verifies grub2's signature and runs it
3. grub2 extracts and loads initramfs into memory, which among other things contains init
4. grub2 verifies the kernel signature and runs it
5. the kernel runs init and the boot process completes

Note that, assuming the hardware hasn't been tampered with, the only vulnerable steps are 3 and 5. This is because all code executed in steps 1, 2, and 4 has been signed using either Microsoft's or Canonical's SecureBoot key. Thus, protecting initramfs from tampering by encrypting the */boot* partition is necessary for a tamper-proof software booting process. For increased security, custom keys may be enrolled in UEFI and used to sign grub2 and the kernel (shim would no longer be necessary).

The following sections provide a bit more detail on each of the steps above, assuming SecureBoot as well as encrypted */boot* and */root* partitions.

## Ubuntu Boot Process (UEFI)

### UEFI & SecureBoot
Most modern computers contain platform firmware for booting which follows the Unified Extensible Firmware Interface (UEFI) specification; this is what mkfeud targets. Even though hybrid disks which boot both in BIOS and UEFI systems can be created [^1], it requires extra complexity (and has no real benefit to myself).

[^1]: https://superuser.com/a/1047690

When the power button on the computer is pressed, the power supply begins supplying power to the computer, and the various chips present in the motherboard go into a default state. At this point, the UEFI will go through several stages[^2], verification of its own integrity, initialisation, selection of the boot device, handover to OS bootloader, and finally handover to OS. During these steps, assuming SecureBoot is enabled, the digital signature of the OS bootloader and the OS are validated before being loaded. In case the signature is not valid, they will not be loaded and booting will fail[^3].

[^2]: https://en.wikipedia.org/wiki/UEFI#Boot_stages
[^3]: https://en.wikipedia.org/wiki/UEFI#Secure_Boot

The public keys used to verify the digital signatures are usually shipped with the computer, with Microsoft being the Certificate Authority in possession of the private keys[^4]. Even though most UEFI systems allow enrolling custom keys, which they can then use to verify self-signed bootloaders / kernels, this is not a simple, user-friendly process[^5]. Thus, shim was developed to facilitate the usage of SecureBoot with many linux distros.

[^4]: https://wiki.ubuntu.com/UEFI/SecureBoot#What_is_UEFI_Secure_Boot.3F
[^5]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_your_own_keys

### Shim
Shim is a simple first-stage bootloader containing embedded keys specific to a linux distribution, which functions as the root of trust for the remaining boot stages by verifying the signatures of following boot programs. This allows Microsoft to sign a distribution's version of shim, delegating the responsibility of signing the rest of the boot programs to the distribution's maintainers[^6]. In most cases, shim will verify and then load grub, as is the case for drives built using mkfeud.

[^6]: https://wiki.ubuntu.com/UEFI/SecureBoot#How_UEFI_Secure_Boot_works_on_Ubuntu

Note that when mkfeud is used to build bootable removable drives, a copy of shim must exist in the UEFI boot fallback path[^7]. This is because a removable drive must be bootable on machines which don't have an NVRAM entry to boot from it.

[^7]: https://en.wikipedia.org/wiki/UEFI#UEFI_booting

**Note:** shim exists at `(hd0,gpt1)/EFI/BOOT/BOOTX64.efi` (the UEFI fallback path) and, if not a removable drive, also at `(hd0,gpt1)/EFI/ubuntu/shimx64.efi`.

### Grub2
When grub2 is loaded, the *core.img* will read the ESP *grub.cfg* configuration file and load any modules needed for the boot process[^8]. Then, grub will request the */boot* encryption password from the user, decrypt the partition, and read the */boot grub.cfg* configuration file. This file contains the menu entries information, like the kernel images and respective initramfs archives. After verifying the selected kernel image, as well as loading both the kernel and initramfs into memory[^9], the kernel will be executed and the boot process continues.

[^8]: https://www.gnu.org/software/grub/manual/grub/grub.html#Images

[^9]: https://unix.stackexchange.com/a/391596

**Note:** grub's *core.img* exists at `(hd0,gpt1)/EFI/ubuntu/grubx64.efi` and *grub.cfg* at `(hd0,gpt1)/EFI/ubuntu/grub.cfg`.

**Note:** it's common to have more than one kernel and respective initramfs for fallback purposes, which exist at `(hd0,gpt2)/vmlinuz*` and `(hd0,gpt2)/initrd.img*`.

### Kernel
After the kernel is loaded, it will extract the initramfs archive (loaded into memory by grub2) into the rootfs[^10]. At this point, if the */root* partition is encrypted, the kernel will use programs in initramfs to decrypt it, either by requesting the encryption password from the user or using a key file. Finally, the kernel checks for the init script within initramfs and runs it as PID 1, which completes the remaining necessary steps to boot the system[^10].

[^10]: https://www.kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt

### Conclusion

If the above achieved it's purpose of providing context on the Ubuntu boot process, you will be confident that (with SecureBoot enabled) the shim, grub2 and kernel code has not been tampered with. However, you may have noticed how initramfs is not signed and thus remains vulnerable to attacks[^11]. By encrypting the */boot* partition, unauthorized access to initramfs is blocked, which should give you full confidence the entirety of the boot process has not been tampered with.

[^11]: https://twopointfouristan.wordpress.com/2011/04/17/pwning-past-whole-disk-encryption/


