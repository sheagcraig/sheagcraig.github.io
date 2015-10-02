---
author: scraig
comments: true
date: 2015-07-16 19:38:24+00:00
layout: post
slug: now-i-get-it-updating-mac-firmware
title: Now I get it... Updating Mac firmware
wordpress_id: 213
---

![Spinal-Tap-Harry-Shearer]({{ site.url }}/images/2015-07-16-now-i-get-it-updating-mac-firmware/Spinal-Tap-Harry-Shearer.jpg)
Perusing [Allister Banks' look into Thunderstrike
vulnerability](https://www.afp548.com/2015/03/05/thunderstrike-need-to-know/)
it didn't quite click for me at first what I was looking at.

The condensed version that I finally was able to tease out was that you can get
yourself into a situation where you:

1. Have some machines with old, unpatched firmware
2. You build an image with AutoDmg or some other method to upgrade those
   machines (in our case, we went from 10.9.5 to 10.10.4 during the summer
   vacation)
3. The machine retains the old, unpatched firmware.

The reason for this is that the FirmwareUpdate part of the combo updater isn't
included in the images built by AutoDMG. If you instead update through the App
Store/SUS, part of the update process includes patching the firmware.

Upon delivering the next point update, the firmware would probably get updated.
Probably.

In terms of Thunderstrike vulnerability, you only need to have a newer firmware
version than as detected in this [extension
attribute](https://gist.github.com/sheagcraig/962b1ec99882b80d03dc#file-thunderstrikevulnerabilityea-py).

So the final word is, if you're imaging machines with out-of-date firmware, and
you care about the firmware being up-to-date, AND you don't have pending point
updates to apply, you can go grab the FirmwareUpdate.pkg package from the most
recent OS X combo updater in your SUS and install it on the affected machines.

You can find this update by looking in your SUS, which is of course Reposado...
{% highlight bash %}
    find /path/to/reposado -name "OSXUpdCombo*"
{% endhighlight %}

...will give you the paths to whatever OS X combo updaters you still have
kicking around. Within (probably the newest one) you'll find the
FirmwareUpdate.pkg you seek.

For example, the 10.10.4 one is here: 
/reposado/html/content/downloads/02/26/031-25777/01sza4ly2cuww3yxfpsbeov51p5n3v7l87/FirmwareUpdate.pkg
    
The package safely does nothing if a machine is already up-to-date, or if the
package is actually too old, so there's no harm in a speculative or
better-safe-than-sorry run (or two).

Hopefully this is an infrequent circumstance. And you could find yourself in a
circumstance where a firmware that ships with a machine is newer than the most
recent OS X update contains. That firmware will probably be a part of the
_next_ update, but you might not want to wait.
