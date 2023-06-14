Title: My strangest Linux install to date

My dad bought an HP EliteBook 820 G2 for cheap. It came without an hard drive, but looking at
the mobo, it supported both 2.5' sata and a M.2 slot. However, the M.2 slot only supported 40mm and 60mm drives.

Moreover, the 2.5' slot needs an adapter from the pins exposed on the mobo to a proper sata
connector, which our laptop is lacking. This meant that I could not use some SSDs I had laying around.

 Anyway, we managed to find 40mm M.2 drives on eBay and we bought one, has the shipping
 required less time that ordering the adapter (at nearly the same cost).

 My dad has some hardware (namely a ScanSnap scanner) that only works on Windows (not supported
	 by SANE yet :/), so once I installed the drive I went
 to install Windows, but I got an error saying "Windows could not be installed on
 this drive", pointing to some BIOS options preventing the drive from being used as a bootable
 device.

 So I went and checked the boot options in the BIOS, and made sure the M.2/NVME option was
 enabled and first in priority. The I tried installing windows again, only to be greeted with
 the same error.

Mh, strange. Let's try with Linux, what could go wrong? I plug in my Ventoy USB stick and boot
Linux Mint. Install process is flawless and finished without errors, so I reboot, eject the
install media to boot into the newly installed OS. Now the laptop is behaving like there is
no OS installed. 

Ok, let's check the boot menu. No EFI partitions found to boot OSes from.

Maybe a bad drive? Bad installation? I boot up the Mint installer again and check the partitions.
Everything is there, no problem. I also tried using the 80mm M.2 NVME drive from my main laptop, again to no avail.

So, apparently the BIOS lists the M.2 NVME drive as a bootable medium, but cannot actually boot
stuff from it. Like, WHAT, WHY? I honestly cannot explain how much this confuses me. [A friend of
mine](!https://alemauri.eu) advanced the hypothesis that the slot also works for M.2 mSata drives, and the BIOS has
drivers for those but lacks drivers for NVME. Makes sense, although I don't have a mSata
drive laying around to test.

BUT, low and behold, the BIOS also lists the SD card as a bootable medium. And in the boot menu I can actually look
for efi files in the SD card.

My next trick takes a page out of old school Raspberry Pi.
Before the newer generations of Pi (namely before the Pi4) came around, Raspberry Pi's didn't natevely support booting from USB, and only
supported booting from the SD card, yet people used USB SDD's as OS drives, only having the SD
Card to launch the bootloader, which then loaded the OS from the USB SDD.

Here's exactly what we do: we install a working bootloader for Linux on the sdcard, point
it to load the OS from the NVME SSD, then I edit `/etc/fstab` to mount the SD Card as the
`/boot` partition, so that updates and changes to the bootloader are seamless and do not require
manual intervention.


Here's what I have done, in order:
    
- Install systemd-bootctl on the SD Card

- Copy the kernel files from /boot to the SD Card

- Update `/etc/fstab` to mount the SD Card as `/boot`

Cool, now Linux actually starts to boot, loading kernel and bootloader from the SD Card (kinda
	slow, but better than nothing I guess). The keywords are _starts to boot_ because it
never gets to the Display Manager, failing to mount some drive listed in `/etc/fstab`, and
throwing me in the initramfs recovery shell.
From here I can check which drives are giving problems. And guess what? the SD Card that cannot be found.

My only idea at this point is that eMMC device support is loaded as a kernel module, at least for how Mint's
kernel is compiled. A quick check with `zcat /proc/config.gz` that I did later confirmed it.

So I edit `/etc/fstab` once and for all to ignore errors relative to `/boot`. This means that
Mint will not mount the SD Card automatically at boot, but I will have to intervene to manually
on something like a kernel update, since systemd-bootctl requires `initramfs-linux and vmlinuz` to
be on the same partition, which needs to be mounted to `/boot`.

Speaking of kernel updates, `update-initramfs` is broken! And this took me a while to figure out.
`update-initramfs` does exactly what the name says. In Mint, kernel files are stored as
`initrd._version_` and `vmlinuz._version_` files. Upon an update, `update-initramfs` puts
the new kernel in `/boot`, recompiles what is needed and makes sure to keep at least one older version, together with
the current one, in that folder. `update-initramfs` then symlinks the newest
files to `initrd` and `vmlinuz` respectevely to avoid messing with grub.cfg everytime
there is an update.

Well, I formatted my sdcard as FAT32, which does not support symlinks! So I disabled them
by placing this line in `/etc/kernel-img.conf`

    do_symlinks = no

This also means that now I have to intervene on every kernel update to make sure systemd-bootctl
is using the right files. Totally not seamless, but at least it works, am I right? No? ( :/
	).

Needless to say, I ended up buying the 2.5' sata adapter to install a SSD like a normal human
being. Though I will still try to install Win10 on the Sata SSD and Linux on the NVME one and
have a solid dual boot. Yet this little adventure made for a fun night of debugging and
tinkering, which always makes me happy.
