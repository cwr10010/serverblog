---
layout: post
title:  "Upgrade To Stretch"
date:   2017-07-15 08:27:00 +0200
categories: webserver debian
---
As it is time to go ahead and my server is no production system it is time to update it to Stretch. I already had a look at what is changed and the main packages in question apart from the kernel are postfix, dovecot, nginx and php whilst php makes me worries most. Additionaly I upgrade the yubico library. But let us go ahead step by step.
First of all I replace all occurences of jessie in /etc/apt/source.list and /etc/apt/source.list.d/* by stretch:

{% highlight sh %}
$ sed -i 's/jessie/stretch/g' /etc/apt/sources.list
$ sed -i 's/jessie/stretch/g' /etc/apt/sources.list.d/*
{% endhighlight %}

Now let us update the system step by step.
{% highlight sh %}
$ apt-get update && apt-get upgrade
{% endhighlight %}
{% highlight sh %}
$ apt-get dist-upgrade
{% endhighlight %}
After the system installation some component got a major version bump and required some fixes here and there. Notably there was PHP which has got bumped up to version 7.0. So I did copy my FPM pool files to the new location for PHP7. I also had to reinstall some PHP modules like APC.
{% highlight sh %}
$ cp /etc/php5/fpm/pool.d/* /etc/php/7.0/fpm/pool.d/
{% endhighlight %}
MariaDB was a bit strange. I had had the MariaDB repository in the source.list and for Stretch the current version for 10.1 was 10.1.23 while Jessy was at 10.1.25 already. So I ran into some version conflicts. Somehow apt also installed libmysqlclient instead of libmariadbclient so PHPs mysqllibrary was unable to load the correct driver which caused me some headaches until I figured that out. 

Finally I had to remove Apache2 again which I ditched in favor of Nginx some time ago. I still don't understand why this dependency was resolved during update.

Postfix got bumped but it dealt with the old configuration quite fine, as well as Dovecot and Nginx. All went fine without problem.

That was basically it! Very view hastle for such a mijor update! Great work from the Debian guys!

