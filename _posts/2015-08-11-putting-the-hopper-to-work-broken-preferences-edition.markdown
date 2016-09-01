---
author: shea_craig
comments: true
date: 2015-08-11 19:24:05+00:00
layout: single
slug: putting-the-hopper-to-work-broken-preferences-edition
title: 'Putting the Hopper to Work: Broken Preferences Edition'
wordpress_id: 223
---

![ecf1b5c2b2d2930afe3e7285ceb0825c]({{ site.url }}/images/2015-08-11-putting-the-hopper-to-work-broken-preferences-edition/ecf1b5c2b2d2930afe3e7285ceb0825c.jpg)
Just a quick one:
Today I was trying to replace a preference file that we install via a package
with a profile. It just didn't work. Restore the plist file and it worked fine.
This mystified me, and initially I thought I was just doing something wrong.
And then I had an inkling of what could be happening.

I tossed the binary for the software-in-question into the Hopper dissambler and
started searching for references to CFPreferences and to the path for the plist
file. Sure enough, I was able to follow through the code and determine that the
software created a proper plist, but that it was manually creating and reading
that plist file; it didn't use the preferences system to _get_ its preferences.

![Gotcha]({{ site.url }}/images/2015-08-11-putting-the-hopper-to-work-broken-preferences-edition/Gotcha.png)
