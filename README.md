HowTo: Add a debian chroot to your Kindle 4NT
Written by max on November 17, 2013 Categories: Debian, Kindle Tags: Debian, Kindle

It’s easier to install debian in a chroot running on the kindle than installing a debian directly to the device. In order to do this, the kindle has to be jailbreaked and a sshd must run.

I decided to create a 500MB image residing in the FAT partition, sharing space with the documents. The image is created on the host (x86) and then copied over to the kindle (armhf). The cross-installation was performed with multistrap.

Disclaimer before you begin: I take no responsibility if you brick your device!
Prepare the image on the host

First create a debroot.multistap.conf:

 

    [General]
    unpack=true
    noauth=true
    cleanup=true
    #debootstrap=Grip
    #aptsources=Grip
    debootstrap=Full
    aptsources=Full

    [Grip]
    # space separated package list
    #keyring=debian-archive-keyring
    source=http://www.emdebian.org/grip
    suite=wheezy-grip

    [Full]
    packages=apt-utils apt iputils-ping net-tools procps
    source=http://ftp.us.debian.org/debian
    keyring=debian-archive-keyring
    suite=wheezy

 

Create an image file with ext3 on it:

    dd if=/dev/zero of=debroot.img bs=1M count=500

    losetup -v -f debroot.img
    mkfs.ext3 /dev/<LOOPDEV>
    losetup -d /dev/<LOOPDEV>

Extract the armhf-packages via multistrap:

    mkdir debroot
    mount -o loop debroot.img debroot
    multistrap -a armhf -d debroot -f debroot.multistrap.conf
    umount debroot

and copy the image to your kindle:

    scp debroot.img <YOURKINDLEIP>:/mnt/us/

Now login with ssh to your kindle..
On the kindle:

This one sets up the chroot (you can run this on every startup):

    mkdir -p /mnt/us/debroot
    mkdir -p /mnt/us/base-debroot
    mount -o loop /mnt/base-us/debroot.img \
    /mnt/base-us/base-debroot/
    fsp /mnt/base-us/base-debroot /mnt/us/debroot -o \
    rw,sync,kernel_cache,max_write=65536,max_readahead=65536
    mkdir -p /mnt/us/debroot/run
    mount -o bind /sys /mnt/us/debroot/sys
    mount -o bind /proc /mnt/us/debroot/proc
    mount -o bind /dev /mnt/us/debroot/dev
    mount -o bind /tmp /mnt/us/debroot/tmp
    mount -o bind /var/run /mnt/us/debroot/run
    mount -o bind /dev/pts /mnt/us/debroot/dev/pts

    mkdir -p /mnt/us/debroot/mnt/orig_fs
    mount -o bind / /mnt/us/debroot/mnt/orig_fs

    chroot /mnt/us/debroot /bin/bash

     

fsp is a filesystem-proxy, which is needed to prevent the kindle from io-locking. I haven’t found the sources anywhere.
Well, we are in the chroot now..

There were issues with dash/bash and I had to execute

    /var/lib/dpkg/info/dash.preinst

 

Now configure the extracted packages:

    dpkg –configure -a

and finally re-login:

    logout
    chroot /mnt/us/debroot /bin/bash

Optionally:

    ln -sf /proc/mounts /etc/mtab
    ln -sf /mnt/orig_fs/etc/resolv.conf /etc/resolv.conf

to umount, after you have killed all processes in the chroot, do:

    umount /mnt/us/debroot/*/*
    umount /mnt/us/debroot/*
    umount /mnt/us/debroot
    umount /mnt/base-us/base-debroot

 
Tell me if it worked for you or if there were problems!

 

Todo: make it read-only to prevent wear out of the flash

Note, that dpkg has a problem installing some packages when using fsp, but without fsp, large operations are leading to lock-ups. You have to disable fsp temporarily for some packages.

For example, procps cannot be installed. From the strace:

    11747 symlink(“skill”, “/usr/bin/snice.dpkg-new”) = 0
    11747 write(2, “D000100: “, 9) = 9
    11747 write(2, “tarobject symlink creating”, 26) = 26
    11747 write(2, “\n”, 1) = 1
    11747 lchown32(“/usr/bin/snice.dpkg-new”, 0, 0) = 0
    11747 utimensat(AT_FDCWD, “/usr/bin/snice.dpkg-new”, {{1349116936, 0}, {1364483015, 0}}, AT_SYMLINK_NOFOLLOW) = -1 ENOENT (No such file or directory)
    11747 write(2, “dpkg: error processing /var/cach”…, 171) = 171
