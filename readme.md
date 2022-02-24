# Openwrt on WD myBookLive Duo (MBLD)

Collected from different sites, to make it a nice RAID1 box again.

DISCLAIMER - ALL YOUR DATA ON THE DISKS WILL BE LOST. most likely...

As many other people i was suprised from Western Digitals message [wd-21008] , but luckily not effected with data loss.  
After some back and forth i ended up with a nice new qnap nas. But damn, what's better than one raid1 NAS? Two, of course.

# stuff you need
1. separate Linux box - i used a raspi i had around - you can also use [etcher] if you are Win only.
1. any USB sata HDD enclosure with external power
1. image from [openwrt-mbl]

# prepararation
This is what i did on my other linux box
1. `wget https://downloads.openwrt.org/releases/21.02.1/targets/apm821xx/sata/openwrt-21.02.1-apm821xx-sata-wd_mybooklive-ext4-factory.img.gz` (check [openwrt-mbl] for a fresh image.)
1. `gunzip openwrt-21.02.1-apm821xx-sata-wd_mybooklive-ext4-factory.img.gz`
1. get the disks from your MBLD

## Disk 1
1. put it into the USB enclosure and attach it to your linux box.
1. `lsblk` to check the drive name - in my case it was `/dev/sda`
1. if there is any data left run `sudo dd if=/dev/urandom of=/dev/sda bs=1M count=2` to wipe it.
1. run `sudo gdisk /dev/sda`
1. then 
    1. `x` to enter experts mode
    1. `z` to zap and Blank out the MBR
1. run `sudo dd if=openwrt-21.02.1-apm821xx-sata-wd_mybooklive-ext4-factory.img of=/dev/sda bs=64k` to flash the drive

1. if you want to save the extra step finding the device on 192.168.1.1 mount the second partition and change/add the following
    1. add/change `../etc/config/system` on the mounted partition
        ```
        config system
            option hostname 'oli-cloud'
        ```
   1. run `../etc/config/network` add/change the following so you do not need to connect to 192.168.1.1 - in this case it puts eth0 on dhcp
        ```
        config interface 'lan'
            option proto 'dhcp'
            option device 'eth0'
            option hostname '*'
        ```

1. unmount 
1. then run `sudo udisksctl power-off -b /dev/sda` to power off properly
1. put this drive in the right slot (where drive B was before) of your MBLD enclosure.

## Disk 2
If this drive is not new, 
1. put it into the USB enclosure and attach it to your linux box
1. ```lsblk``` to check the drive name - in my case it was `/dev/sda`
1. if there is any data left run `sudo dd if=/dev/urandom of=/dev/sda bs=1M count=2` to wipe it. (maybe even that is not enough...)
1. then run `sudo udisksctl power-off -b /dev/sda` to power off properly

Put this drive in the left slot (where drive A was before) of your MBLD enclosure.

# Getting serious
1. connect your MBLD and power it up,
1. if you skipped adding the `../etc/config/network` file, you'll find it after a couple of minutes on 192.168.1.1
    - you can use your browser to update the hostname, use the ip you like and apply. Set a password if you like.
    - or you can connect via ssh to the box and run `uci set system.@system[0].hostname=somevalue` followed by `uci commit system` to set the hostname
1. if you changed the ip, go find your machine again.

## Disk 1
The right disk received the /dev/sda address.
Connect via ssh, then  
1. run `opkg update` will update the package list
1. run `opkg install gdisk blkid e2fsprogs` to install gdisk
1. run `gdisk /dev/sda`
1. gdisk will come up with an errormessage, that your GPT is damaged
1. then
    1. `1` to use the MBR
    1. `r` to get into the recovery menu
    1. `f` to build a fresh GPT from the MBR and confirm (this will change the disklabel / partitiontype to gpt)
    1. `m` to go back to the main menu
    1. `n` to create a new partition
    1. `3` to make it the third one
    1. let the first sector be `2097152` to leave space for OpenWRT changes
    1. take the default as last sector
    1. `fd00` as partitiontype Linux RAID autodetect
    1. `p` to validate
    1. `w` to write the partitions and confirm
1. if gdisk complained that the kernel still uses the old partitions, run `partx -a /dev/sda` and ignore the message.

## Disk 2
The left disk received the /dev/sdb address.
1. run `gdisk /dev/sdb`
1. then 
    1. `n` to create a new partition
    1. `1` to make it the first one
    1. let the first sector be `2097152` to leave space for OpenWRT changes
    1. take the default as last sector
    1. `fd00` as partitiontype Linux RAID autodetect
    1. `p` to validate
    1. `w` to write the partitions and confirm
1. put this drive in the left slot (where drive B was before) of your mybook live duo enclosure

## create the RAID1
If everything went well so far `lsblk` should look like this

```
root@oli-cloud:/etc/config# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  2.7T  0 disk
├─sda1   8:1    0    8M  0 part /boot
├─sda2   8:2    0  104M  0 part /
└─sda3   8:3    0  2.7T  0 part
sdb      8:16   0  2.7T  0 disk
└─sdb1   8:17   0  2.7T  0 part
```

everything about mdadm on [man-mdadm].  
1. validate that there are no traces of any former raids nowhere with running
    - `mdadm --examine /dev/sda /dev/sdb` to check the disks
    - `mdadm --examine /dev/sda3 /dev/sdb1` to check the partitions  
    You don't want to see any superblock.
1. run `mdadm --create /dev/md0 --level=mirror --raid-devices=2 /dev/sda3 /dev/sdb1` to start the raid creation

    alternative: where you could maybe sneak around wiping 2 disks at once. Not fully validated as i synced them before having the md0 formatted.
    ```
    mdadm --create /dev/md0 --level=mirror --force --raid-devices=1 /dev/sda3
    mdadm --manage /dev/md0 --add /dev/sdb1
    mdadm --grow /dev/md0 --raid-devices=2
    ```

1. it will take a while...
1. you can run 
    - `cat /proc/mdstat` to check the progress or 
    - `mdadm --detail /dev/md0` to get more details or
    - `mdadm --examine /dev/sda3 /dev/sdb1` to get more more details

I needed to wait that to finish. All other tries to reboot when not synced ended in misery...
When it's done `lsblk` should look like this:
```
root@oli-cloud:~# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda       8:0    0  2.7T  0 disk
├─sda1    8:1    0    8M  0 part  /boot
├─sda2    8:2    0  104M  0 part  /
└─sda3    8:3    0  2.7T  0 part
  └─md0   9:0    0  2.7T  0 raid1 /mnt/raid
sdb       8:16   0  2.7T  0 disk
└─sdb1    8:17   0  2.7T  0 part
  └─md0   9:0    0  2.7T  0 raid1 /mnt/raid
```

## Format and mount
1. then format the partition using `mke2fs -t ext4 /dev/md0`
1. `mkdir /mnt/raid` to create a nice mountpoint
1. `mount -v -o sync /dev/md0 /mnt/raid`
1. `umount -v /dev/md0`

## make the raid persistent
run `mdadm --detail /dev/md0 | grep UUID` to get the uuid. 
Or run `mdadm --detail --scan --verbose` to get a line looking like this
```
ARRAY /dev/md0 level=raid1 num-devices=2 metadata=1.2 name=oli-cloud:0 UUID=3923a9d6:df6b0d4b:d1db476e:cf791c02
   devices=/dev/sda3,/dev/sdb1
```
Then you need to edit `/etc/config/mdadm` to have that raid persistent. The options follow the tags in [man-mdadmconf].  
Use the received values to make nice config file, like i did here:
```
config mdadm
        option email root
        # list devices /dev/hd*
        # list devices /dev/sd*
        # list devices partitions

config array
        option uuid 3923a9d6:df6b0d4b:d1db476e:cf791c02
        option device /dev/md0
        option name oli-cloud:0
        option level raid1
        option metadata 1.2
        list devices /dev/sda3
        list devices /dev/sdb1
        option spares 0
```
Then run `uci commit mdadm` to validate and see if it works.

## fstab
details on fstab [openwrt-fstab].  
To mount the device with evey reboot, you need to get the right UUID and change the fstab
1. run `blkid | grep md0` to get the UUID of the '/dev/md0'. you need to replace that in the following commands
```
uci add fstab mount
uci set fstab.@mount[-1].target='/mnt/raid'
uci set fstab.@mount[-1].uuid='61c5c6dd-ca98-4228-8f03-c234500cafbd'
uci set fstab.@mount[-1].enabled=1
uci set fstab.@mount[-1].enabled_fsck='1'
uci set fstab.@mount[-1].options='rw,sync,noatime,nodiratime'

uci commit fstab
```

# optional packages
1. run `opkg update` to update the package repos

## Samba Server
1. run `opkg install samba4-server sambe4-client luci-app-samba` to install smb and nfs shares

## hd-idle 
details on [openwrt-hd-idle].  
run `opkg install hd-idle luci-app-hd-idle` to control when the hdds goes into sleep
- go to "Services" > "HD-Idle" in the browser page to configure
- or run 
```
uci set hd-idle.@hd-idle[0].enabled='1'
uci set hd-idle.@hd-idle[0].idle_time_interval='10'
uci add hd-idle hd-idle
uci set hd-idle.@hd-idle[-1].disk='sdb'
uci set hd-idle.@hd-idle[-1].idle_time_interval='10'
uci set hd-idle.@hd-idle[-1].enabled='1'
uci set hd-idle.@hd-idle[-1].idle_time_units='minutes'
/etc/init.d/hd-idle enable
/etc/init.d/hd-idle start
uci commit hd-idle
```

## WOL
details on WOL on [openwrt-bs-wol] and [openwrt-wol].  

## Performance
details on removing kernel netfilters on [openwrt-bs-netfilters].  
```
mkdir /etc/bak
mv /etc/modules.d/nf-* /etc/bak/
mv /etc/modules.d/ipt-* /etc/bak/
```
needs a reboot.

## S.M.A.R.T
details on [openwrt-smartmontools].  
run `opkg install smartmontools smartmontools-drivedb` 

## LED
details on [openwrt-leds].  
code taken from [braian87b].  
```
uci del system.@led[-1]

uci add system led
uci set system.@led[-1].name='Alive'
uci set system.@led[-1].sysfs='mbl:blue:power'
uci set system.@led[-1].default='0'
uci set system.@led[-1].trigger='timer'
uci set system.@led[-1].delayon='1000'
uci set system.@led[-1].delayoff='500'

uci add system led
uci set system.@led[-1].name='DiskUsage'
uci set system.@led[-1].sysfs='mbl:green:power'
uci set system.@led[-1].default='0'
uci set system.@led[-1].trigger='disk-activity'

# Additional if we want to manually turn on red Led on some Error.
uci add system led
uci set system.@led[-1].name='Error'
uci set system.@led[-1].sysfs='mbl:red:power'
uci set system.@led[-1].default='0'
uci set system.@led[-1].trigger='default-on'

uci commit system
```

## rsync
details on [man-rsync]  
how to use [do-rsync]  
run `opkg update; opkg install rsyncd` to install

## NFS
details on setting up nfs on [openwrt-nfs].  

## UART
Nice article from mybookworld on [mbw-uart].  

# Troubleshooting

## constanlty flashing green led after reboot and not getting online
Never figured this one really out. Mostlikely the WD bootloader found some stuff from the old disks and gone crazy.  
Only fix was start from the beginning.

## raid not there
to add it manually run

1. `mdadm --assemble /dev/md0 /dev/sda3 /dev/sdb1` to rebuid the raid.
2. `mount /dev/md0 /mnt/raid` to mount it.

But it should work with the correct settings in the /etc/config/mdadm file.

## remove the raid
properly removing or cleaning up a raid is not that simple.  
Remove a drive [raid-02]  
Removing metadata from drives [raid-03]  
Replacing hard disks [raid-04]  

# useful resources

## mdadm and config
manpage for mdadm [man-mdadm].
manpage for mdadm.conf [man-mdadmconf].  
pull request with some more details on the `/etc/config/mdadm` file [openwrt-pr4700].  
other unfinished raid1 guide on [openwrt-sh-raid]. 

## Update OpenWRT on the MBLD
About upgrading: [openwrt-mbl-upgrade]  
About mising partitions: [openwrt-mbl-partitions]

## OpenWRT general
About the OpenWRT UCI and config files: [openwrt-uci] [openwrt-configfiles]
About Logging and `logread` [openwrt-logs]

## Raid on Linux
About the raid setup [kernel-raid-setup]  
About Linux raids [kernel-linux-raid]
Creating a raid with MBR [raid-01]  

## OpenWRT and MBL(D)
braian87b installing on a MBL [braian87b]  
ewaldx installing on a MBL [ewaldx]  
forum thread about MBLD [openwrt-16195]  
forum thread about MBLD [openwrt-86632]
forum thread about extroot [openwrt-20157]

<!-- resources -->
[openwrt-mbl]:          https://openwrt.org/toh/western_digital/mybooklive  "OpenWRT MyBook Live"
[openwrt-mbl-video]:    https://www.youtube.com/watch?v=RneEa9fZGqM "OpenWRT MyBook Live on Youtube"
[openwrt-mbl-upgrade]:  https://openwrt.org/toh/western_digital/mybooklive#upgrading "OpenWRT MyBook - Live Upgrading"
[openwrt-mbl-partitions]: https://openwrt.org/toh/western_digital/mybooklive#recreating_missing_partitions "OpenWRT MyBook Live - Missing Partitions"
[openwrt-leds]:         https://openwrt.org/docs/guide-user/base-system/led_configuration#networkactivity "OpenWRT Led Configuration"
[openwrt-nfs]:          https://openwrt.org/docs/guide-user/services/nas/nfs.server
[openwrt-smartmontools]: https://openwrt.org/docs/guide-user/additional-software/smartmontools
[openwrt-uci]: https://openwrt.org/docs/guide-user/base-system/uci
[openwrt-configfiles]: https://openwrt.org/docs/guide-user/base-system/uci#configuration_files
[openwrt-hd-idle]: https://openwrt.org/docs/guide-user/storage/hd-idle
[openwrt-fstab]: https://openwrt.org/docs/guide-user/storage/fstab
[openwrt-logs]: https://openwrt.org/docs/guide-user/base-system/log.essentials
[mbw-uart]: http://mybookworld.wikidot.com/wd-mybook-live-duo-uart  
[man-mdadm]: https://linux.die.net/man/8/mdadm
[man-mdadmconf]: https://linux.die.net/man/5/mdadm.conf
[man-rsync]: https://linux.die.net/man/1/rsync
[openwrt-pr4700]: https://github.com/openwrt/openwrt/pull/4700/files
[raid-01]: https://www.linuxbabe.com/linux-server/linux-software-raid-1-setup  
[raid-02]: https://delightlylinux.wordpress.com/2020/12/22/how-to-remove-a-drive-from-a-raid-array/
[raid-03]: https://serverfault.com/questions/488346/removing-raid-metadata-from-drives
[raid-04]: https://www.howtoforge.com/replacing_hard_disks_in_a_raid1_array
[braian87b]: https://gist.github.com/braian87b/84802005e61f7080620b2d522b804f0d  
[ewaldx]: https://github.com/ewaldc/My-Book-Live  
[openwrt-20157]: https://forum.openwrt.org/t/extroot-with-raid1/20157/4  
[openwrt-16195]: https://forum.openwrt.org/t/solved-wd-mybook-live-duo-two-disks/16195
[openwrt-86632]: https://forum.openwrt.org/t/raid-1-on-a-my-book-live-duo-project-fun/86632
[openwrt-bs-netfilters]: https://openwrt.org/toh/buffalo/ls421de?s[]=raid#remove_kernel_netfilters
[openwrt-bs-wol]: https://openwrt.org/toh/buffalo/ls421de?s[]=raid#power_off_and_wol
[openwrt-sh-raid]:https://openwrt.org/toh/shuttle/kd20?s[]=raid#raid1_installation_process_under_construction
[openwrt-wol]: https://openwrt.org/docs/guide-user/advanced/auto_wake_on_lan
[kernel-raid-setup]: https://raid.wiki.kernel.org/index.php/RAID_setup  
[kernel-linux-raid]: https://raid.wiki.kernel.org/index.php/Linux_Raid  
[wd-21008]: https://www.westerndigital.com/support/product-security/wdc-21008-recommended-security-measures-wd-mybooklive-wd-mybookliveduo  "WD 21008"
[etcher]: https://www.balena.io/etcher/ "Etcher"
[do-rsync]: https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories-de  "rsync tutorial"