Jeff Cassell 2026FEB19


How to manually create a persistent Ubuntu boot disk :)


# Intro

   All of these instructions are for the terminal and use Linux tools, but I
   tried to explain everything well enough that you could translate it to a GUI
   or another app/operating system if needed. I also wrote much of this during
   my interminable insomnia so there's bound to be some weird formatting
   somewhere that I missed during editing, so prepare your delicate, discerning
   eyes.

   I include definitions here and there where I felt like it was necessary but I
   tried not to overdo it to keep things as simple as possible. Here's a little
   quasi table of contents for the steps involved:

   ### Overall Steps

   1. Mount all the partitions within the ISO

   2. Partition the destination disk

   3. Copy the ISO's root partition to the destination disk's

   4. Populate the ESP partition (boot partition)

   ### LEGEND

   ``` text ```     A command to execute
   `text`           A specific program
   "text"           Just some quoted, regular old text
   <text>           A required parameter (don't include the brackets <>)
   [text]           An optional parameter (again, don't include the brackets [])

   I'm not sure I even used this last one (optional parameter), but just in
   case, that's what it means (same as in every Linux documentation ever).



# Mount All the Partitions within the ISO

   First, let's get a few terms defined (simply) so I'm not speaking French and
   things make a little more sense in your own head as we go along. Do some
   googling if you need more detail, but this should do for now. It's also not
   strictly necessary, so if you like you can skip this reading-intensive part
   and come back if you get confused/curious. I personally think it's important,
   so do with that information what you will. These are terms that describe a
   hard drive and all the relevant parts.

   Mount:
   Attach a device to the system so that it can be interacted with by the user.
   It is being "mounted" to an existing part of the system.

   Disk:
   Readable/writable media. Can be an old hard disk drive (HDD), a new solid
   state disk (SSD), or a USB flash drive. Things like that.

   Partition:
   A portion of a disk. Meant to hold a file system within it. A single
   partition can take up the entire disk, but a computer's main disk usually has
   several, because there are other partitions that need to be squeezed on there
   for everything to work right on the computer. Alone, without a file system, a
   partition is just an empty container. There can exist empty space between
   partitions for purposes of disk longevity and other optimizations, and the
   free space doesn't hurt anything. Although there have been some very sneaky
   viruses that have taken up residence in little spaces like this...

   File system:
   A way for the computer to actually read/write *files* to a partition. Yep,
   that's it. Easy! Think of it as another container, this time within the
   partition (itself a container), but this takes up the *entire* partition and
   isn't divided into any smaller pieces. This can store an operating system
   (it's all files), or it can be data backed up by the user (more files),
   pretty much whatever you need. It's all just files as far as the computer is
   concerned. But a file system is absolutely vital to perform any work that is
   meaningful to the user on the other end of the keyboard. People like to use
   files/directories as opposed to hexidecimal memory addresses and length
   values, which is just a tiny piece of what's happening when you save a new
   file to disk.

   Operating system (OS):
   The stuff we all know and love (or hate). Windows, Mac OS X, Linux, etc. This
   can only exist when there is a file system to write it to disk, and a
   partition to hold that file system.

   So, to summarize:

   Disk -> Partition 1 -> File system -> Bootloader (ignition to start car)
        -> Partition 2 -> File system -> Operating system (car to drive)
        -> Free, unused space

   For each of these things (partition, file system, OS), there are different
   systems, or *flavors*, that exist, such as there being Windows and Linux for
   OSs. Some have a bigger ecosystem and others a smaller one.  You don't need
   to worry about those details for this, just understand that, as of now, there
   is no single partition system, or file system, or (obviously) OS that rules
   them all. Sadly. Maybe some day in the future that will be the case, and
   we'll all use the metric system, and roads will be paved with candy. But
   alas. Not yet.

   The Ubuntu ISO contains several partitions (at least as of Ubuntu 24.04.3
   LTS), and we want to ignore all but one of them: the root partition, where
   Ubuntu actually resides. This is where "/" lives, and all the accompanying
   structure below it (i.e., "/home/jeff" is below the "root", that initial
   starting slash "/" in front of "/home/jeff").

   We need to find out the size of this partition so we can copy it
   byte-for-byte without making a "close-enough" sized partition that wastes
   space. It's not strictly necessary, but this will save every possible byte of
   disk space, which I'm personally a fan of. Plus I believe in fundamentals,
   and this establishes a very strong foundation.

1. Mount the ISO to a loopback block device

   Loopback device:
   A logical (i.e., purely virtual and doesn't actually exist) device so that
   the computer can interact with something when it otherwise wouldn't be able
   to.

   Block device:
   To skip a lot of technical jargon, you can usually think of block devices as
   a synonym for disks, like hard drives or USBs.

   Finally, we can fire up a Linux terminal and get started. We're going to
   start by mounting all the partitions within the Ubuntu ISO to our file system
   so we can interact with them.

   ``` sudo losetup -fP --show <ubuntu_ISO> ```

   This runs `losetup` (loopback setup) to first find (-f) an available loopback
   device on the operating system, scan the new device's partition table for
   partitions (-P), and then show the loopback device after it's been created
   (--show). It should return something like "/dev/loop6" when finished. Keep
   note of this. I'll try to consistently refer to it as the ISO loopback
   device.  Now we need to locate its root partition and determine the exact
   number of disk sectors it occupies.

   Sector:
   A group of bytes that the disk physically reads/writes at a time. These
   groups of bytes are contiguous, meaning they're all right next to each other
   and not spread around randomly on the disk. Because the disk HAS to write in
   these chunks of bytes, it's the smallest size that something can occupy on
   the disk, and this size varies from disk type to disk type (USB, HDD, SSD,
   etc.). This size is permanent and physically tied to the disk. To illustrate,
   USBs usually have a sector size of 512 bytes, and that means if you write a
   file to the disk that is smaller than 512 bytes, you're actually wasting a
   teeny tiny bit of space, and if you write something larger then you take up
   multiple sectors. It is possible to fill a disk with tiny files and waste a
   lot, lot of space, but in reality this pretty much never happens unless some
   asshole writes a virus to do it.

   File systems also have their own "chunks" that they write called "blocks"
   which are a multiple of 512 bytes, like 2048 or 4096, which makes that
   hypothetical disk-destroying tiny files situation even more pronounced.
   
2. Get the Exact Size of the Root Partition

   Now we need to locate the root partition. Use ``` lsblk ``` (list block) and
   look for the loopback device that was created in the previous step. It should
   be a tree structure, with the loopback device at the top of the tree and its
   individual partitions as the branches. The root partition will be the
   largest; several gigabytes, denoted by a "G" at the end of the partition's
   described size.  Mine is "/dev/loop6p1" with a size of 5.9G, so the root
   partition is partition 1 (denoted by the "p1").

   ``` sudo parted <ubuntu_ISO_file> unit s print ```

   This will run `parted` (partition editor) on the Ubuntu ISO *file* (not the
   loopback device) and change the described unit size to "disk sectors" then
   print the partition table of the ubuntu ISO. You can also run this agains the
   loopback device, both will work, I just chose the ISO file itself for reasons
   unknown.

   Anyway. ***Note the size of the root partition (partition 1 for me); mine is
   "12383424s", or 12,383,424 sectors.*** This will be important when we're
   partitioning the disk in a little bit. Also make sure the sector size is
   "512B/512B" (listed above Partition Table) because we'll compare this later
   with the physical disk when we're writing. It shouldn't be an issue, but it
   doesn't hurt to be careful.  If it's anything else, you need to add an extra
   option to `dd` later.



# Partition Your Destination Disk

   This is the bulk of the work we're doing. I've mostly been
   describing/explaining things up to now, but, once you're past this
   step, it's all downhill.

1. Make sure the disk is partitioned and empty

   Obviously, backup anything you need to because we're going to wipe this USB
   disk.

   ``` lsblk ```, to find your USB drive. Should be "/dev/sda", or "/dev/sdb".
   Something along those lines. Look at the size of the drive to determine
   exactly which it is. Mine is "/dev/sda", and I know that because the size is
   "59.5G" or about 60 gigabytes (my USB says 64GB on it but that's a lie
   manufacturers tell people because they count a kilobyte as 1000 bytes, when
   they're actually 1024 -- yeah :o). Now let's wipe the destination disk.

   ``` sudo wipefs -a <USB_device> ```

   Again, my USB device was "/dev/sda". Also note that this is not a complete
   wipe of the disk, it just removes the important bits that tell the disk
   what lives where. All the data is still on the disk. This is fine, this is
   what normally happens when you "delete" something from your disk. To really,
   really remove something, you have to overwrite it, which is overkill unless
   you're intentionally trying to hide something. Now let's open the USB device
   to create the new partition table for our empty disk.

   Partition table:
   A table that describes to the disk where each of its partitions are, how big
   they are, and if they have any special flags that the computer might care
   about for special handling. In order to create any partitions, first this
   table needs to exist so that those partitions' details can be saved to it.
   It's kind of like an address book.

   ``` sudo gdisk <USB_device> ```

   This will open the `gdisk` terminal (GPT format disk). You can type "?" if
   you're curious about all the commands. Use ``` o ``` to make a brand new
   partition table.

   Now let's make sure the sector size is 512 bytes with ``` p ```, listed under
   "Sector size". This is standard for USB flash drives, so unless this is an
   external hard drive we should be fine.

2. Boot/ESP partition

   ESP:
   EFI System Partition. This is the partition that the computer will look
   in when booting to try to find a bootloader. Bootloaders are just programs
   that get the ball rolling when starting the computer.

   From within `gdisk`, use the command ``` n ``` to create a new partition.  It
   will prompt for the partition number, defaulting to 1. The default is fine.
   Accept the default first sector, which is the first usable (and
   speed-optimized) sector of the disk. For the last sector of the partition,
   use ``` +100m ``` to specify 100MB, which should be plenty for the bootloader
   later, as well as any bootloader(s) in the future.

   Set the partition type code to ``` ef00 ```, which is shorthand for a very,
   very long GUID code that will automatically get filled in on the partition.

3. Root partition

   This will be the actual file system that we're going to be using, i.e.,
   Ubuntu, and where it's going to live. It will be read-only since this is a
   fixed image. It's called the "root" partition because the root directory,
   "/", is the root of the file system that everything else is attached to.

   This is where we need the size of the root partition that we found earlier!
   Way back in step 2 of "Mount All the Partitions of the ISO."

   From within `gdisk`, create the root partition with ``` n ``` once again, and
   accept the default for the next partition number (which should be 2).  Again
   accept the default for the starting sector, and for the last sector we'll use
   ``` +<size_in_sectors> ``` from what we found earlier (mine was 12383424).
   The default type code is fine.

4. Persistence partition

   This will consume the remainder of the space on the disk and is where all the
   changes we make to the "read-only" file system will live.

   From within `gdisk`, create the last partition with ``` n ```, accept the
   default partition number, accept the default starting sector, and accept the
   default ending sector (the end of the disk). The default type code is fine as
   well.

   FINALLY, we can write all these changes to our disk with ``` w ```.
   Afterwards, If you want to check out all the work we just did, use ``` lsblk
   ``` again and check out your drive now (recall that mine was "/dev/sda"). It
   should now have the nice new tree structure with 3 partitions, each perfectly
   sized. :)

5. Write the necessary file systems to each partition

   Remember that a partition can't really do much on its own, it needs a file
   system for the OS to be able to read/write to it. For the boot
   partition, we're going to use FAT32 file system, which we can create with:

   ``` sudo mkfs.fat -F 32 <USB_boot_partition> ```

   My boot partition was "/dev/sda1".

   For the root partition we don't need to do anything, we're just going to copy
   that all over from the ISO, which is a byte-for-byte copy that *includes* the
   file system. Neat.

   For the persistence partition, we're going to use "ext4", which is standard
   fare for modern Linux.

   ``` sudo mkfs.ext4 -L casper-rw <USB_persistence_partition> ```

   My persistence partition was "/dev/sda3". The "-L" applies a file system
   label of "casper-rw", which Ubuntu looks for when enabling persistence.



# Copy the ISO's Root Partition to the destination Disk's

   Now we're going to copy the root partition from the ISO to the USB's root
   partition that we created that's the exact same size. While we are copying a
   specific partition that lives in the ISO, we're not copying it from the file,
   but from the loopback device that it's mounted to. It's the only way we can
   access that partition specifically while ignoring the rest of the image.

   ```
   sudo dd if=<loopback_root_partition> of=<disk_root_partition>
   status=progress
   ```

   Recall that my loopback device's root partition is "/dev/loop6p1", and my USB
   disk's root partition is "/dev/sda2". This copies data (`dd` means something
   like "data dump") from the input file (if=) to the output file (of=), i.e.,
   from the loopback's root partition to the disk's root partition.  This is
   going to take a while, depending on the speed of your disk and everything in
   between.

   When that's done, we can remove the loopback device (my ISO looback device
   was "/dev/loop6"). We're removing the *entire* device, not a specific
   partition.

   ``` sudo losetup -d <ISO_loopback> ```



# Populate the ESP Partition (Boot Partition)

   When booting from a disk, UEFI will look for this partition to start the
   whole boot process off. From here, it will locate the bootloader, which does
   exactly what it sounds like (loads the necessary boot processes).

   Bootloader:
   The starting process that *starts* the bootup of an operating system. The
   bootloader will usually let you choose different options to boot into
   different modes of an OS, such as Windows' "Safe Mode".  The default
   bootloader file that UEFI will look for is BOOTX64.EFI.

   We're going to create the bootloader and its configuration file so that we
   can customize its functionality to our preferences.

1. Install GRUB

   GRUB:
   GRand Unified Bootloader. Linux's bootloader that plays nice with
   multiple other operating systems. Super basic but reliable and customizable.
   Usually looks for a config file (grub.cfg) for further options/instructions,
   automatically selecting the default option after a timeout period (10-30
   seconds, usually).

   To install GRUB, we need to be able to write to the USB's ESP partition,
   which means we need to mount it (i.e., attach it) to our current file
   structure within Linux. My ESP partition was "/dev/sda2".

   ``` sudo mount <ESP_partition> /mnt ```

   This will mount the partition to the "/mnt" directory, which usually exists
   by default for this exact purpose. If not, you can mount it to another
   directory that you have read/write privileges. If it's not empty, it won't
   delete what's in there, but it won't be accessible while the partition is
   mounted. Next, we'll install GRUB.

   ```
   sudo grub-install --target=x86_64-efi --efi-directory=/mnt --removable
   --bootloader-id="Ubuntu <your_version>" --uefi-secure-boot
   ```

   If it complains about it, you can remove the "--uefi-secure-boot" option, it
   just means you'll need to disable Secure Boot on the motherboard to boot from
   the disk. The "--target=x86_64-efi" option tells GRUB what platform we need
   it to work with, which is almost always 64-bit x86 CPU architecture with UEFI
   motherboard firmware, unless it's a Mac or something unusual. "--removable"
   is for USBs and UEFI and bundles a lot more modules into the bootloader image
   so it's more portable. "--bootloader-id" should display at the UEFI menu when
   selecting which device to boot from, but depends on the UEFI firmware whether
   or not it uses it.

2. Edit the Config File

   For this, I'm just going to paste a sample config and a single field needs to
   be filled. To get the requisite data for the field:

   ``` lsblk -f ```

   The field we need is "UUID", for the root partition. Mine is
   "2025-08-05-18-20-26-00". This is the filesystem's Universally Unique ID. Now
   let's fill that into our sample config below.

   The only other things you might need to change are "timeout" and "default",
   if you like. Timeout is how long before the prompt will automatically choose
   an entry if we don't choose one ourselves, and default is which entry it will
   choose (starting from 0).

   ```
   set timeout=5
   set default=0 
   set root=<root_UUID>

   menuentry "Ubuntu 24.04.3 LTS with Persistence" {
      linux /casper/vmlinuz --- quiet splash persistent
      initrd /casper/initrd
   }

   menuentry "Ubuntu 24.04.3 LTS" {
      linux /casper/vmlinuz --- quiet splash
      initrd /casper/initrd
   }
   ```

   This is going to go into "grub.cfg", which should be in
   "/mnt/EFI/BOOT/grub.cfg". Then unmount the partition:

   ``` sudo umount /mnt ```

   And that should be it! Restart and boot from the newly-created disk.

