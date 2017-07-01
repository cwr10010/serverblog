---
title: "Harddrive Replacement is here"
layout: post
date:   2017-07-01 16:40:00 +0200
categories: homeserver replace harddrive
---
About two weeks ago the harddrive of my home server failed. I wrote a [post][post_harddrive-failure] about this. Yesterday Western Digital delivered the replacement. So it was time to put it into the server. As I wrote I have replaced the failed drive already. So this will extend the RAID by one adding about 3TiB.

To start the replacement I add the drive to the fourth bay and start the server. As the SATA controller of the server is set to AHCI the drive is detected as an additional drive and so will be managed by the Linux kernel.

After booting into Linux I repeat the same first steps of the last article where I have to copy the partition table and the UEFI partition of the first drive. Next I add the second partition to the RAID 1 as a spare and the third as a swap partition.

On to business. I now need to add the partition for the RAID-5 drive.

{% highlight sh %}
$ mdadm --add /dev/md1 /dev/sdd4
{% endhighlight %}

This alone makes the drive available as a spare drive but I want to extend the RAID. So I need to grow it. This will issue a reshape of the array and take about 20 hours. So I return next day to proceed.

{% highlight sh %}
mdadm --grow /dev/md1 --raid-devices=4
{% endhighlight %}

After those 20 hours I extend the physical LVM volume. But first I want to know the size of the resized RAID-5 volume.

{% highlight sh %}
$ mdadm -D /dev/md1
/dev/md1:
        Version : 1.2
  Creation Time : Sat Oct 17 00:43:07 2015
     Raid Level : raid5
     Array Size : 8747888640 (8342.64 GiB 8957.84 GB)
  Used Dev Size : 2915962880 (2780.88 GiB 2985.95 GB)
   Raid Devices : 4
  Total Devices : 4
    Persistence : Superblock is persistent

  Intent Bitmap : Internal

    Update Time : Sat Jul  1 18:29:26 2017
          State : clean
 Active Devices : 4
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 0

         Layout : left-symmetric
     Chunk Size : 512K

           Name : attila:1  (local to host attila)
           UUID : XXX
         Events : 144267

    Number   Major   Minor   RaidDevice State
       3       8        4        0      active sync   /dev/sda4
       1       8       20        1      active sync   /dev/sdb4
       2       8       36        2      active sync   /dev/sdc4
       4       8       52        3      active sync   /dev/sdd4
{% endhighlight %}

This displays the meta data of the device. You can also see the history of the RAID here. The first is number 3 which I did put in 2 weeks ago. The last is the new one today. Number 0 was from the drive I have sent to WD. Now, in the future I will have about 8.2TiB for the LVM volumes. So I extend the pyhsical volume to that size.

{% highlight sh %}
$ pvresize /dev/md1
{% endhighlight %}

I have decided to just extend the /home partition. It is 2TiB at the moment and will be about 4.5TiB after the resize.

{% highlight sh %}
$ lvresize -l 100%VG /dev/vol0/home
{% endhighlight %}

The -l parameter tells lvresize to extend to 100% of the physical VolumeGroup. See `man lvresize` for more. Now the device has the 4.5 TiB available but the partition is just at about 2 TiB. The filesystem also has to be extended. I use resize2fs for that. Normaly I need to unmount the partition but it also works online (with a risk). So I stop all services and issue the command

{% highlight sh %}
$ resize2fs /dev/vol0/home
{% endhighlight %}

This takes some seconds as it formats the added space. After that the full space is available.

{% highlight sh %}
$ df -h
Filesystem             Size  Used Avail Use% Mounted on
udev                   3,9G     0  3,9G   0% /dev
tmpfs                  794M   26M  769M   4% /run
/dev/md0               9,1G  2,3G  6,4G  27% /
tmpfs                  3,9G     0  3,9G   0% /dev/shm
tmpfs                  5,0M  4,0K  5,0M   1% /run/lock
tmpfs                  3,9G     0  3,9G   0% /sys/fs/cgroup
/dev/mapper/vol0-tmp   9,2G   21M  8,7G   1% /tmp
/dev/mapper/vol0-var   9,1G  833M  7,8G  10% /var
/dev/mapper/vol0-srv   3,6T  1,7T  1,9T  48% /srv
/dev/mapper/vol0-home  4,6T  549G  4,0T  12% /home
{% endhighlight %}

With that I am done. The drive is added and running fine ever since.

_FIN_.

[post_harddrive-failure]:{% post_url 2017-06-18-002-harddrive-failure %}
