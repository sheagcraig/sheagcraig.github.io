---
author: scraig
comments: true
date: 2014-12-09 14:33:39+00:00
layout: post
slug: zipping-non-flat-packages-for-casper
title: Zipping Non-Flat Packages For Casper
wordpress_id: 18
categories:
- jssimporter
- python-jss
---

Just a short little technical note. python-jss just added support for the Jamf
distribution server as a distribution point type, which hopefully will make
both python-jss and JSSImporter (I'm going to start calling it that instead of
jss-autopkg-addon) useful for more people. And just to let people know the
direction for the future, I'm hoping to add CDP support very soon.

Anyway, AutoPkg makes flat packages, which makes sense since non-flat packages
have been deprecated for quite awhile now. Non-flat packages still work, and
indeed, in our shop we don't bother to do anything special to handle them.
However, you can't PUT or POST a directory of files—you have to archive
multiple files into a single file or have some kind of mechanism for multiple
uploads. Casper Admin's solution to this is to zip the package prior to
uploading (if you have an incompatible DP configured). You can see that the pkg
gains a .zip extension. On the policy side, the jamf binary knows what to do:
it unzips the package prior to installing.

But some vendors are still shipping non-flat packages (Silverlight is the
example I've been testing with).

Github user MitchellSBlake pointed out to me that this is often needed not only
for JDS distribution points, but also for SMB shares to work around packages
that may have broken symbolic links.

After trying a number of different approaches, I've decided to take a couple of
steps to continue to work with non-flat packages:
- .ZIP/.zip as an extension is now considered a package
- The JSSImporter will test for whether a package is non-flat, and
  automatically zip it up into the same directory that the package came from
  (probably as reported by pkg_path

A couple of further implications: ALL non-flat packages will get zipped. You
don't have to have a JDS for this to occur. So there will probably be some more
processor overhead, especially as larger packages are compressed. If this
starts to be a memory issue or causes people issues, I can drop back to not
actually compressing the packages. It also means that JSSImporter is going to
want to re-upload any packages that you may already have on your distribution
points, as it now is in a zip.

Also, python-jss will just fail if you try to copy a package to a JDS; however,
it will issue a warning that tells you why it won't work. It's up to you figure
out how you want to handle this. Please take note, that if you created a
Package object before uploading, only to discover that your package was not
flat, you'll probably want/need to change the name to match the new .zip
extension.
