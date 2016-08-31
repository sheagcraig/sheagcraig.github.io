---
title: "Clean up Your Filthy Repo: Part 1 (Names)"
date: 2016-08-30T12:05:25-05:00
comments: true
layout: post
---
![On my honor, I promise to do my best!]({{ site.url }}/images/2016-08-31-clean-up-your-filthy-repo/boyscout.jpg)

### Look at you. You're a Mess
I don't know about you, but I've got a lot going on. I get the impression that
most Mac admins are flying solo, and there's always more to do. Also, I like my
things to be tidy. As a child, I would sit on the floor of my room and
fantasize about the ultimate layout that would solve all of my lego-display and
8-bit gaming needs. Likewise, while it often has no visible impact on my
end-users, I'm constantly scheming about reorganizing and tidying up my digital
assets.

At a previous job I managed a Casper environment. Because I'll take any excuse
to write code to solve a problem, I wrote a tool called "Spruce" to help me
with both major organizational busywork, and day-to-day structural cleanliness
auditing.

Now I'm using Munki to manage my fleet, and I wanted the same ability to keep
things tidy without a lot of terminal faff. So I started writing a Spruce for
Munki admins.

Of course, that's a bit revisionist; what actually happened is that I wrote a
script here and there to do different things, mostly to help me gain greater
understanding and insight into the structure of the organization and its
management as I settled into my new job. At a certain point, it made sense to
share the work I had done with others and create a single unified interface for
it. Thus, Spruce.

Anyway, I like the boy scout approach to code espoused by Robert C. Martin:
Every time you take a look at a piece of code, return it cleaner than you found
it (Paraphrased). There are always opportunities to fix formatting, improve
style, and update or improve documentation, let alone refactoring or other
work.

Similarly, for long stretches of time, Munki just kind of ticks along on its
own without too much intervention. AutoPkg fills the repo with patches, I
rotate things through testing into production with another automated tool
(coming soon), and I munkiimport the occasional Xcode Beta for our developers.
But when I do get in there and start working directly with the repo, I try to
improve it and look for inconsistencies, miscategorizations, etc.

Spruce makes this way easier.

So this blog post is the first of many where we're going to look at ways you
can use Spruce to oil, torque, and maintain the running parts of your
management tool.

### Getting Spruce. Getting it Running.
Clone or download the current [Spruce on GitHub](https://github.com/sheagcraig/spruce-for-munki).

Enter the spruce directory and run

{% highlight Bash %}
$ ./spruce name
{% endhighlight %}

Since this is your first time, Spruce will offer to walk you through
configuring its preferences to point to a Munki repo. It will use your
munkiimport preferences if they are in the usual place. Mount the share to your
repo, and step through the configuration prompts. When mounted, my repo is just
"/Volumes/munki_repo".

You may want to put spruce in /usr/local for use later. You can alias or
link the spruce executable so it's available from anywhere.

### Getting Dirty
Once you've configured Spruce, it will continue with the `name` command.

{% highlight Bash %}
# ... snip!
ADPassMonExtras
ADPassMonProfile
AdobeAIR
AdobeFlashPlayer
AdobeReader
AdobeReaderDC
AmazonEC2Tools
AnyConnect
Atom
AutoDMG
AutoPkg-Release
AutoPkgr
Box Sync
BrotherPrinterDrivers
# ...
{% endhighlight %}

Spruce tore through your pkginfo files and collected the `name` values into a
set (A set ignores duplicates). From this, we can quickly see all of the items
we offer, and look for mistakes, similar but slightly different items, etc.

For example, say you came across the following:
{% highlight Bash %}
# ... snip!
GitHub Desktop
Go
GoToMeeting
Google Chrome
GoogleChrome
GoogleDrive
GoogleEarth
GoogleEarthPro
GoogleTalkPlugin
# ...
{% endhighlight %}

Right away, my human eye, developed over millenia to spot patterns, notices the
doubled Google Chrome entry. I would want to go look and see why I have both a
"Google Chrome" and a "GoogleChrome" in the repo. Possible reasons for this
could be that my Chrome AutoPkg recipe had been overridden at one point (the
official recipe uses "GoogleChrome"), a coworker may have inadvertently added
one manually, or there could be old packages that predate AutoPkg + MunkiImport
usage.

Regardless of the reason, this is a potential problem. A manifest that
specifies "Google Chrome" is not going to get the same software offered that it
would if it had "GoogleChrome" specified.

Spruce has tools to automate standardizing the `name` which we'll look at in
the future.

Another helpful thing that you can do is search your repo's names. Try the
following:

{% highlight Bash %}
$ ./spruce name google
Google Chrome
GoogleChrome
GoogleDrive
GoogleEarth
GoogleEarthPro
GoogleTalkPlugin
{% endhighlight %}

Spruce does a case-insensitive search for your search term across all of the
names and reports back. I use this all the time when I can't remember exactly
what `name` I use for a particular item that I want to add to a manifest. It's
a little quicker to use than trying to remember how to do a multiline awk that
results in just the names. For example, say I did want to add a
`managed_update` for Google Chrome, and I couldn't remember whether there was a
space or not in the name. Running the above command quickly answers that
question for me.

### And now, for my final trick
Add a `-v` and the output changes a bit. Now, Spruce lists, for each `name`,
the versions of that software currently in the repository:

{% highlight Bash %}
$ ./spruce name -v GoogleChrome
GoogleChrome
           44.0.2403.155
           44.0.2403.157
           45.0.2454.85
           45.0.2454.93
           45.0.2454.99
           45.0.2454.101
           46.0.2490.71
           46.0.2490.80
           46.0.2490.86
           47.0.2526.73
           47.0.2526.80
           47.0.2526.106
           47.0.2526.111
           48.0.2564.97
           48.0.2564.103
           48.0.2564.109
           48.0.2564.116
           49.0.2623.75
           49.0.2623.87
           49.0.2623.110
           49.0.2623.112
           50.0.2661.75
           50.0.2661.94
           50.0.2661.102
           51.0.2704.63
           51.0.2704.79
           51.0.2704.84
           51.0.2704.106
           52.0.2743.82
{% endhighlight %}

The `-v` flag works both for global name output and searches. It provides you
with a quick way to see how many versions you're currently archiving (and can
potentially remove). I often use it with no search term to get an overall view
of what products AutoPkg has been pumping out updates for. For example, the
above snippet shows me both that the Chrome developers have been super
release-happy _and_ that I haven't bothered to clean up my Chrome packages for
awhile.

### So long, for now
Check back soon for more Spruce fun.
