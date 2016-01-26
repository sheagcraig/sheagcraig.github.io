---
title: Configuring and Reconfiguring the Default Mail Reader. Self Service through Munki, and a Tale of Woe
date: 2016-01-26T12:05:25-05:00
comments: true
layout: post
---
![Pabst Blue Ribbon!]({{ site.url }}/images/apple_mail_reader/pbr.jpg)

### Introduction
Like many institutions, our supported email client is Microsoft Outlook. Over
the years, a not insignificant number of support-cases have been initiated over
the confusing configuration of a default mail reader on OS X. Specifically,
when users click a `mailto://` link on a webpage, rather than opening Outlook
as they might expect, OS X opens the default mail handler, Apple Mail. The
problem becomes compounded when Apple Mail refuses to let you do anything until
you have successfully configured a mail account through it's new account
wizard. Enough users have a hard time figuring out how to scroll text windows
without grabbing the side-sliders that expecting them to go rooting around in
the preferences is a tall order, especially when we can set it for them.

This post will cover how to set the default handlers for your client machines,
and importantly, also how to then set them back again, which may look simple
after you have solved the first problem, but be wary! Looks can be deceiving!

### TL;DR
You can grab all of the bits and pieces from these two gists:
[Set Microsoft Outlook as Default Handler for mailto, vcf, and ics](https://gist.github.com/sheagcraig/16e9d6a01406de06c524).
[Munki/Outset/PyObjC Self Service Set Apple Mail as Default Handler for mailto]().

### Heavy Support Load: The Problem
Unlike some OS's, where a system preference pane allows you to configure all of
the default handlers for different file types and URL schemes, Apple scatters
them throughout their bundled apps. The default mail reader is set in Mail's
preferences; Calendar in Calendar, contacts in Contacts... etc.

However, this preference is not _set_ in the preference domain for Mail,
Calendar, or Contacts; it shows up in LaunchServices, although setting default
handlers by writing to LaunchServices's preference file with `defaults` or
editing a plist is not the supported means, nor does it seem to work correctly.

Apple provides a LaunchServices framework, however, for managing default
handlers for URL schemes and file extensions. This is a bit more involved than
just running a commandline utility, but it's not terribly complicated either.

Therefore, savvy admins have been making using of the C program duti to
configure their client machines to use non-Apple handlers. For example, see
[Rich Trouton's solution](https://derflounder.wordpress.com/2015/12/01/setting-microsoft-outlook-as-the-default-application-for-email-contacts-and-calendars-via-automator/).

When I began looking at configuring my fleet to use Outlook, I downloaded the
source for duti and tried to compile it myself (there are no binaries on
duti.org or GitHub). This failed, of course, since it hasn't been updated for
10.11. This was easy to fix, but it's when I began to smell the oncoming "This
isn't going to be as quick as I initially thought" smell that was wafting off
of the still smoking binary I had just linked.

For those wanting to build duti, it's perfectly functional to just grab the
current code and build it on a pre-10.11 machine. I think Rich told me his was
built on 10.5!

I wrote a three line script to use duti to set the mailto handler (a URL
scheme) to Outlook, and to also configure Outlook for ics (Calendars) and vcf
(contacts). I set up a pkginfo to deploy duti to my machines, as well as
another pkginfo to execute the above script that `requires` duti.

Without going into extensive details about the testing and release process,
this solution worked great, and I moved on to the next thing. Client machines
built and had the correct app configured for the communications tasks intended.

### Grumbling Out of Earshot: The Next Problem
Then I started to hear from the die-hard Apple users about how they were both
disgusted at my complicity in Microsofting their Apple, and more importantly,
their inability to undo what I had done and use Apple Mail as their default
mail reader. Apple Mail of course still worked correctly; the specific problem
was that trying to use Apple Mail's preferences to set the default mail reader
to Apple Mail failed. After setting the default mail handler to Mail, clicking
on a "mailto" link still opened Outlook, and upon examining the setting in the
Apple Mail preferences, the setting had reverted back to Outlook.

![Frank]({{ site.url }}/images/apple_mail_reader/frank.jpg)

I immediately double-checked all of the configuration profiles, scripts, and
other stuff that comprises our build process. The reverting seemed to occur
about 10 seconds after setting the preference, which seems to point towards
CFPrefsd restoring something.

A combination of `system_profiler SPManagedClientDataType` and `mcxquery` is a
good start to getting a look at managed preferences. This, however, didn't turn
up anything aside from the above duti script.

Next I took a closer look at duti's source, to see if it was doing anything
strange or unexpected. You can see where URL schemes are set
[here](https://github.com/moretension/duti/blob/master/handler.c#L100)

duti makes a single LaunchServices function call to set the handler, and I
didn't see anything else going on.

Next I dug into the LaunchServices
[documentation](https://developer.apple.com/library/mac/documentation/Carbon/Reference/LaunchServicesReference/#//apple_ref/c/func/LSSetDefaultHandlerForURLScheme).
To eliminate duti from the mix, I spun up a Python interpretor and did the
following:

```
>>> import LaunchServices
>>> LaunchServices.LSSetDefaultHandlerForURLScheme("mailto", "com.apple.mail")
0
>>> LaunchServices.LSCopyDefaultHandlerForURLScheme("mailto")
"com.apple.mail"
```
...and I was thinking to myself, "fine, I'll just do it with python. And then
to be safe I repeated the last command...
```
>>> LaunchServices.LSCopyDefaultHandlerForURLScheme("mailto")
"com.microsoft.outlook"
```

It had reset on me while I was prematurely congratulating myself for a job
well-done.

I set up an infinite while loop to poll the mailto handler every 5 seconds and
tried a number of things. What I discovered after a number of tests was that
not only could I not set the default mail handler back to Mail in any way (duti
or the GUI), but also that I could recreate this problem without ever using
duti, or the python/PyObjC above. I was able to recreate it with _just_ the
Apple Mail preferences. Sure enough, with a little bit of Googling, I found a
thread where people were complaining about this very problem.

Just to confirm, I threw Apple Mail into Hopper and could verify that it wasn't
using any super-secret private methods or anything; it was using the exact same
function duti and my python were using.

### setDefaultEmailHandler

```
void -[DefaultApplicationPopUpButton _setDefaultEmailHandler:](void * self, void * _cmd, void * arg2) {
    rdx = arg2;
    r14 = self;
    r12 = _objc_msgSend;
    rbx = [[self selectedItem] retain];
    r15 = [[rbx representedObject] retain];
    [rbx release];
    if (r15 != 0x0) {
			# Here it is!
            rcx = LSSetDefaultHandlerForURLScheme(@"mailto", r15);
            if ((rcx != 0xffffd5bb) && (rcx != 0x0)) {
                    rdx = rcx;
                    NSLog(@"Error setting %@ as the default email handler: %d", r15, rdx);
            }
    }
    rax = (r12)(r14, @selector(indexOfSelectedItem), rdx, rcx);
    (r12)(r14, @selector(setIndexOfSelectedHandler:), rax, rcx);
    rdi = r15;
    [rdi release];
    return;
}
```

After a lot of fiddling around, I learned a few more things:
- Only the "mailto" handler seems to be an issue. The vcf and ics stuff worked
  back and forth without issue.
- Once you set a mail handler _once_ through any means, that handler is now set
  _forever_!
- However, I discovered that you can set it, and then immediately log out, and
  the new setting is retained. If you dilly-dally, the preference gets set
  back, and you're stuck on whatever the first changed default handler was.

I have filed, and have had closed as a duplicate, a bug with Apple about this
issue, so they know about it, and it will probably go away soon. So while the
specific need to allow users to reset their mail handler to Apple Mail will go
away, I think the solution I came up with serves as a good example of how to
make use of Munki's OnDemand feature and PyObjC to add descriptive and helpful
self service items to Managed Software Center. Casper Admins can of course
modify the following to make use of their Self Service feature without too much
work.

### How I Did It
At this point I had already been setting and getting my mail handlers with
Python, so I decided to drop using duti. This saves me from an added dependency
on my client machines.

Then, I have two sets of pieces:
Setting Microsoft Outlook as the Default Handler for mailto, vcf, and ics:
- Munki pkginfo to install...
	- Package built with the Luggage.
		- Outset login-once python script to set handlers.

Setting Apple Mail back:
- Munki OnDemand pkginfo to install...
	- Package built with the Luggage.
		- Outset OnDemand python script.
			- Presents a popup dialog explaining what is about to happen.
	- Postinstall script to touch Outset's OnDemand trigger.

The initial configuration is installed using a Configuration manifest that is
applied to all client machines. Setting the mail handler back to Apple Mail is
placed in a Self Service manifest, along with a category of Self Service, to
help users find it. It has our corporate logo, and a detailed description of
the process to supplement the dialog box that the script will present. Really,
this is the easy part.

Here are the interesting parts of the initial configuration. See the
[full gist](https://gist.github.com/sheagcraig/16e9d6a01406de06c524) for all of
the pieces.

{% gist sheagcraig/16e9d6a01406de06c524 set_outlook_default_handler.py %}
{% gist sheagcraig/16e9d6a01406de06c524 set_outlook_default_handler.pkginfo %}

Setting the mail handler back after you've set it the first time is a little
more involved, mostly because we don't want to just kill loginwindow like a
savage. I had timed Alert class already lying around from [this
project](https://github.com/sheagcraig/auto_logout), so I just grafted it in,
even though I don't make use of the timer.

The logout is executed, strangely enough, by shelling out to Applescript,
sending the «event aevtrlgo» event to loginwindow. This allows us to skip the
"Are you sure you want to quit all applications and log out now?" dialog.
However, things like active terminal sessions will still block the logout,
thus, the explanatory text requesting the user to prepare for logout. While
it's of course very easy to just drop the user to the loginwindow, or do a
forced logout, I prefer to give the user more control of what is happening
given that this is a self service environment. The auto_logout project
mentioned earlier demonstrated that rebooting rather than a forced logout
avoided some weird graphics glitches that resulted from killing loginwindow
(which immediately gets respawned), with the same result (user ends up at the
login window). However, having to logout to set the mail handler seemed silly
enough; rebooting just seemed like a travesty.

As before, here are the interesting parts; see the [full gist](https://gist.github.com/sheagcraig/4fa2a11b7f1738e11c79)
for the complete set of pieces.

{% gist sheagcraig/4fa2a11b7f1738e11c79 set_apple_mail_default_handler.py %}
{% gist sheagcraig/4fa2a11b7f1738e11c79 set_apple_mail_default_handler.pkginfo %}
{% gist sheagcraig/4fa2a11b7f1738e11c79 postinstall %}
