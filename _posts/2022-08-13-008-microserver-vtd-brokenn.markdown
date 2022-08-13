---
layout: post
title:  "VT-D is broken on HP Microserver Gen8 on Linux Kernel 5.15"
date:   2022-08-13 15:11:00 +0200
categories: homeserver ubuntu linux microserver
---
Some weeks ago I migrated my HPE Microserver Gen8 to the current LTE release of Ubuntu, namely 22.04. Right after the start I noticed error messages regarding memory mapping and missmatched IDs regarding IOMMU and DKE. After some research I found out that with the current stable Ubuntu Kernel some Gen8 and Gen9 servers from HPE are having problems with virtualization and especially VT-D. One solution was to switch off intel iommu management entirely for that. This can be done by adding intel_iommu=off to the GRUB kernel parameters.

{% highlight sh %}
GRUB_CMDLINE_LINUX_DEFAULT="nomdmonddf nomdmonisw intel_iommu=off"
{% endhighlight %}

But this did not solve the problem entirely. I had continuous problems of interrupt failures. The server all the sudden became unresponsive via network and created errors while writing to disk - something you definitely do not want on a NAS. THe final solution was to switch off VT-D in the servers BIOS. This deactivates direct access to hardware through the virtualization layer. But as I do not run VMs on that thing this is an OK solution for me. 

As a ressource in the Ubuntu bug tracker this is the ticket for reference: [Ubuntu Bug Tracker](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1970453){:target="_blank"}{:rel="noopener noreferrer"}
