---
layout: post
title:  "Harddrive Failure"
date:   2017-06-18 06:38:00 +0200
categories: homeserver failure harddrive
---
So here comes the reason why I started to write this BLOG. I know that i forget alot of things over time and that problems return in non frequent manner. The problem of today is a failing hard drive. This is not unusual and happens especial when a server runs 24/7. This hard drive used to run in my home server. It is one out of three [3TB WD RED][wd-red-3tb]{:target="_blank"} drives and the one I added when I moved from a SoHo NAS to the [MicroServer][microserver]{:target="_blank"}. It has a different model number than the other two.

First signs of failure I had about a month ago when the drive dropped out of the RAID-5. Interestingly I did not have any information in the log. So I added it back and hoped for the best. About a week ago my sever did its monthly raid resync which took unusual long. So I checked and found complaints about not readable sectors in the log written during the resync. What struck me was that I did not have any warnings from the SMART daemon that should send me messages when soemthing fishy is up with the hard drives. So I ran `smartctl -a /dev/sda`. And yes there was a sector marked as unrecoverable and serveral others recoverable while reading. Time to replace.

As the hard drive is still under warranty I have contacted [Western Digital][wd-support]{:target="_blank"} and they will send me a replacement the next days. Thanks for this uncomplicated and unexited support! To be protected anyway I also bought a replacement at the local MediaMarkt. For better or worse it has the same model number as the one that was failing. The drive that comes back from WD will be drive number 4 in the RAID-5 then. Until now the server was running just on the minimal set of three drives. I also decided not to use hardware RAID capabilities of the server and stick to software RAID as I can swap controller more easily and I am free in designing the partitioning.

To understand the procedure of replacement I have to describe the setup of the server a bit further. As I wrote the server currently houses three hard drives of 3TB each. All drives share the same GPT partition scheme except that drive two and three lack the uefi partition. But the partitions start at the same sector everywhere. By the way the failing disk is the first and the one that I boot the system from. Maybe it is time to create the boot partition on the other disks as well, just in case.

{% highlight sh %}
Number  Start   End     Size    File system     Name  Flags
 1      1049kB  512MB   511MB                   uefi  bios_grub
 2      512MB   10,5GB  10,0GB                  root  raid
 3      10,5GB  14,5GB  4000MB  linux-swap(v1)  swap
 4      14,5GB  3001GB  2986GB                  lvm   raid
 {% endhighlight %}

The `uefi` boot partition contains all boot information. We do need that to allow grub to boot the kernel. The `root` partition is part of a RAID-1 volume. Until the replacement of the first drive the third served as a spare here. After I replace the first drive the spare volume will be on this drive as drive three kicked in. The root partition has to be RAID-1 as grub can not boot RAID-5 volumes. Next partition is the `swap` partition. I do not create a raid for that and let the swap driver handle the volumes. Every disk has a 4000MB partition for that. With three drives this makes roughly 12GB, increased to 16GB once the fourth drive found its place.

Finally partition four houses the `LVM` partition being part of a RAID-5 volume built upon all three drives. LVM provides logical volumes for `home`, `srv`, `var` and `tmp`. But as I use LVM on top of the RAID I do not need to take care of those during replacement. The rebuild of the RAID will just run on the LVM volume.

{% highlight sh %}
$ mount | grep ext
/dev/md0 on / type ext3 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mapper/vol0-tmp on /tmp type ext2 (rw,nosuid,nodev,noatime,block_validity,barrier,user_xattr,acl,stripe=128)
/dev/mapper/vol0-home on /home type ext4 (rw,relatime,quota,usrquota,grpquota,stripe=128,data=ordered)
/dev/mapper/vol0-var on /var type ext4 (rw,relatime,stripe=128,data=ordered)
/dev/mapper/vol0-srv on /srv type ext4 (rw,relatime,quota,usrquota,grpquota,stripe=128,data=ordered)
{% endhighlight %}

As the stage is set now we start with the replacement of the disk. But hold your horses. You should **not** do that late at night and after the joy of a good irish whisky.

As the fourth drive bay of my home server is free I have mounted the unused brackets to the new drive and inserted the hard drive into the server. Attention, this server is not capable of hot-swaping drives. HP cripled the firmware to protect their higher level servers. I do not need that anyway so no complaint here. You just have to be aware that the server has to be _turned off_. And another word of warning: do not shuffle around the disks in the bays. They are identified by a unique identifier and you will confuse the RAID driver. Do always and only replace one drive!

After booting the server I have copied the partition scheme from the failing disk to the new one:

{% highlight sh %}
$ sfdisk -d /dev/sda | sfdisk /dev/sdd
{% endhighlight %}

This works as the hard drives are of equal size (in fact they are identical in their specs). Keep in mind that smaller disks will cause trouble as the partitions will not fit on it. So you should always replace disks of the same brand and model but with different model numbers. To bad that I have got one with the same model number of the drive that was faining.

Next I did need to copy the `uefi` boot partition  to the new disk. I have used the `dd` tool for that:

{% highlight sh %}
$ dd if=/dev/sda1 of=/dev/sdd1 bs=1M
{% endhighlight %}

The new disk should be able to boot into the system now. To check if this all worked fine I have shut down the server, replaced the failed disk with the new one in its tray and trigered a reboot. Fingers crossed the system booted fine into the system. Time to re-add the partitions.

I issued a
{% highlight sh %}
$ mdadm /dev/md0 --add /dev/sda2
{% endhighlight %}

to add the volume to the RAID. As stated above, by removing the old disk from the arrays the spare volume from disk 3 became the second active mirror in the RAID-1 and the new disk will from now on provide the spare volume. For the LVM volume (don't forget to adapt the partitions names in the shell command) this will trigger the rebuild of the array which takes some hours. As the CPU is very busy during that time and the rebuild also takes a lot of I/O the server showed not to be very responsive.

The final step was to add the swap partition.  First tell the partition that it will funktion as such:

{% highlight sh %}
$ mkswap /dev/sda3
{% endhighlight %}

Second add it to the swap volume:

{% highlight sh %}
$ swapon /dev/sda3
{% endhighlight %}

This ends the work that I had to do. Pretty easy, isn't it? Of course your mileage may vary depending on the failure state of your disk. If you are unable to copy the uefi partition, you are in more serious trouble. That is also a reason why I should very soon add the uefi partition to the other drives.

_FIN_.

[wd-red-3tb]: https://www.wdc.com/de-de/products/internal-storage/wd-red.html#WD30EFRX
[microserver]: https://www.hpe.com/de/de/product-catalog/servers/proliant-servers/pip.hpe-proliant-microserver-gen8.5379860.html
[wd-support]: https://support.wdc.com/warranty/warrantystatus.aspx?lang=de
