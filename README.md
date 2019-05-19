# zram-boot-setup

## Just Why?

1. I was bored
2. My RAM Usage basically never exceeds 8 GB, leaving 24 GB idling around pretty much all the time
3. Firefox pops up really fast with this setup
  
## And How?

By using the linux kernel module zram and the initcpio mechanism from arch linux

### zram

zram is a kernel module which provides block devices whose data is actually compressed and put in RAM, so basically a compressed ramdisk.

Usually, zram might be used as swap; the pages which usually reside in memory are well compressable, I think the description
of the kernel module said something like "2:1 compression"; given 4G RAM, you use 3G normally, and have a zram-swap with 2G
in size, only taking up the remaining 1G of RAM.  
Downsides of zram include a cpu-overhead for every read/write operation.  
More information on its more intended use-case is in the [improving performance topic from the arch linux wiki](https://wiki.archlinux.org/index.php/Improving_performance#Zram_or_zswap).

### initcpio

mkinitcpio is used in arch linux to create the intial ramdisk. To quote from [man 8 mkinitcpio](https://manned.org/mkinitcpio.8) (the same is quoted on the [arch linux wiki page](https://wiki.archlinux.org/index.php/Mkinitcpio)):

> The initial ramdisk is in essence a very small environment (early userspace) which loads various kernel modules
> and sets up necessary things before handing over control to init. This makes it possible to have, for example,
> encrypted root file systems and root file systems on a software RAID array. mkinitcpio allows for easy extension
> with custom hooks, has autodetection at runtime, and many other features.

### Putting it together

So I simply wrote a hook for mkinitcpio which creates a zram device, configures it and copies all data from an actual partition on a hard drive to it:

* [`/etc/initcpio/install/ram_arch`](./etc/initcpio/install/ram_arch)
* [`/etc/initcpio/hooks/ram_arch`](./etc/initcpio/hooks/ram_arch)
* [`/etc/mkinitcpio.conf`](./etc/mkinitcpio.conf)

Of course, the boot loader needs to know what block device should be mounted as the root filesystem; an example entry
for the systemd-bootloader is also in this repo:

* [`/boot/loader/entries/02-arch-ck-zen-ram.conf`](./boot/loader/entries/02-arch-ck-zen-ram.conf)

### Special to my setup

I did all this as a test/experiment on a system which already has a few OSes running splendidly on it, especially an
existing arch linux installation, whose bootloader I reused for this. To slim down the chances of messing up the
existing installation, I had to do some extra steps:

* Giving a different name to the .img-File generated by mkinitcpio. Not hard to do, see
  [`/etc/mkinitcpio.d/linux.preset`](./etc/mkinitcpio.d/linux.preset)
* Unless needed, the shared boot-partition is mounted read-only, see [`/etc/fstab`](./etc/fstab)

## Does it work?

Yes it does.

The copying process during boot time has the added benefit of going back to the good ol' days of turning on your PC
and having time to go get a coffee.

### Minor Inconveniences

* Long boot time
  ```
  # systemd-analyze 
  Startup finished in 14.921s (firmware) + 2.987s (loader) + 1min 57.496s (kernel) + 6.954s (userspace) = 2min 22.360s 
  graphical.target reached after 7min 44.309s in userspace
  ```
* Installation process: Well, you can run pacman just like always, yes, but everything you do will be gone with the
  next shutdown. I wrote [`/usr/local/sbin/mchr`](./usr/local/sbin/mchr) which first runs the given command normally,
  and, if successful, then tries to chroot to the original hard drive root filesystem and run it there too.  
  This, of course, includes everything that may want to persist stuff to the disk, like network configuration, user and
  group management, systemd service changes (e.g. enable), a.s.o.
* Aside from system stuff, unless you have an extra home partition, everything written there will be lost too. That's the
  main reason I didn't install Thunderbird on this setup.

### Are there upsides?

* Not once in my life did I see the firefox window open this fast
* That, of course, applies to other bigger/complex software as well
* The "usually everything is not persistent" part __might__ be seen as a security benefit; if a virus or attacker messes up
  your system (without mounting the hard drive root filesystem ofc), you simply reboot (and get a fresh coffee).
  Going a step further, one could physically disconnect the hard drive as soon as the copy process has finished.

## Q&A

### Do you recommend setting up a system like this for actual usage?

__NOPE__

### I would like to try this, just for the lolz. Can I use the stuff in this repo as-is?

__NOPE__: Steps like preparing a partition, installing arch on it a.s.o. are not covered here, and likely will
look different for every person. In this repo, everything is hardcoded, nothing configured. Use at your own risk,
and only if you know what you're doing.
