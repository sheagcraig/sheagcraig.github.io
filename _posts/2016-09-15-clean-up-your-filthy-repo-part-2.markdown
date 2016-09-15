---
title: "Clean up Your Filthy Repo: Part 2 (Out of date stuff)"
date: 2016-09-15T10:00:00-05:00
comments: true
layout: single
---
![WuTang]({{ site.baseurl }}/images/2016-09-15-clean-up-your-filthy-repo/36th-chamber.jpg)

### I'll let you try my Wu-Tang Style

Picking up where we left off, it's time to do something a little more fancy.

All of my software patches are automated with AutoPkg. This creates, over time,
a lot of unused pkgsinfo and pkgs files in my repository. Drive space seems to
be of no real concern, however, I like to only keep things active in my
repository that I am actually using. I think this goes back to the general
philosophy mentioned in the last post of leaving something a little better than
when you checked it out. Certainly, it makes cloning the entire repository a
good deal faster if you're managing multiple servers and provisioning new ones.

Spruce has a number of reports that we'll be looking at over the next several
posts, but I think that the most immediately useful one is the "Out of Date
Items Report". We will also talk about the corresponding `deprecate` verb and
it's `--auto` flag.

### What is not in use?

![Clan of the White Lotus]({{ site.baseurl }}/images/2016-09-15-clean-up-your-filthy-repo/dimmak.jpg)

Spruce uses the same code to provide a report on out-of-date items for both the
`spruce deprecate --auto` and `spruce report` features. Before we get into
usage, let's talk about how it works.

Spruce can determine which items are required by your current production
machines and help you clean up and remove out-of-date ones. This post discusses
the `auto` mode. If you want to pick and choose, you can generate an
out-of-date report in plist format using `spruce report -p`, and then edit that
plist to only remove the items you want to remove using `spruce deprecate -p`.
The ultimate in control.

When Spruce looks through your repo, it builds a dependency tree by first
looking through all of the manifests. This brings up an important point: if you
have manifests that are out-of-date, not used, or reference things you really
don't use, you should get them out of here. That being said, the worst case
scenario is that Spruce's out-of-date "protects" items that it need not protect
from removal.

Each manifest is parsed, and a set is built of items that are used for any type
of manifest item (`managed_installs`, `managed_uninstalls`,
`optional_installs`, etc, and even these types within conditionals).

Next, Spruce parses all of the pkginfo files in the repository. I made a
decision early on that this feature should only look at items with the catalog
"production". My feeling is that this is a pretty practice to denote items that
are available for all clients. Anything not in production is by definition some
kind of "testing" item. Items in testing should be excluded from automated
cleanup until they're either rejected as unsuitable and removed manually, or
promoted to production.

With all of the pkginfo information and all of the manifest information set,
Spruce builds a list of "used" items. This includes every single pkginfo that
matches a name used in a manifest. This is used later to calculate what is
out-of-date.

Spruce then builds the dependency tree. It starts with the first item in the
list of items used in manifests. It then looks through all items that match
that item name and adds the most recent one and a configurable number of
next-most-recent ones to a list of "current" items. (`spruce report` uses only
the most recent item for each dependency, `spruce deprecate --auto` is
configurable; I usually keep 3 deep. Also, only 3 are kept if there are 3 to
begin with!).

As Spruce adds in these items, it checks to see if they have any `update_for`
or `requires` dependencies. If it does, it will check resolve those
dependencies in the same manner, keeping the most recent item that resolves
that dependency. Of note, this includes handling for pinning a dependency with
a version number. For example:

{% highlight Xml %}
<key>requires</key>
<array>
	<string>Sal_Config-1.0.5</string>
</array>
{% endhighlight %}

...will specifically select the 1.0.5 version of Sal_config to consider as
current.

Once all of the manifest items and dependencies have been built (including the
configurable number of "backup" versions), Spruce does set subtraction between
the "used" list and the "current" list. What is left are the out-of-date items.
This thus ignores the testing items, and the items which are not actually in
use by any manifests (there is a report for that which we'll look at in another
post).

### Enough Talk!

![Training]({{ site.baseurl }}/images/2016-09-15-clean-up-your-filthy-repo/Shaolin.jpg)

To begin cleaning up out-of-date items, make sure your repo is mounted, and
then run `spruce report | more` and page through the results, of which there
are many, looking for the "Out of Date Items Report". (One of the features in
the works are ways to specify specific reports to run so you don't have to run
the entire suite every time if you're only interested in one).

{% highlight Bash %}
./spruce report
Out of Date Items Report:
        This report collects all items which are in the production catalog, but
        are not the current release version. Items that have dependencies to
        current releases through either the `requires` or `update_for` keys are
        excluded. Items in non-production catalogs are also excluded from
        consideration by this report.

        Items:
        --------------------
        name: AdobeAIR
        path: /Volumes/munki_repo/pkgsinfo/Digital Media/AdobeAIR-21.0.0.215.plist
        version: 21.0.0.215
        size: 58.20M
        --------------------
        name: AdobeFlashPlayer
        path: /Volumes/munki_repo/pkgsinfo/Digital Media/AdobeFlashPlayer-22.0.0.192.plist
        version: 22.0.0.192
        size: 17.74M
        --------------------
        name: AdobeFlashPlayer
        path: /Volumes/munki_repo/pkgsinfo/Digital Media/AdobeFlashPlayer-21.0.0.242.plist
        version: 21.0.0.242
        size: 17.70M
        --------------------
        name: AdobeReaderDC
        path: /Volumes/munki_repo/pkgsinfo/Productivity/AcroRdrDC_1501620039_MUI-15.016.20039.plist
        version: 15.016.20039
        size: 147.96M
        --------------------
        name: AdobeReaderDC
        path: /Volumes/munki_repo/pkgsinfo/apps/Adobe/AdobeReaderDC-15.010.20060.plist
        version: 15.010.20060
        size: 142.92M
        --------------------
        name: AnyConnect
        path: /Volumes/munki_repo/pkgsinfo/AnyConnect-4.2.01035.pkginfo
        version: 4.2.01035
        size: 12.92M
        --------------------
        name: AnyConnect
        path: /Volumes/munki_repo/pkgsinfo/apps/AnyConnect-4.2.00096.pkginfo
        version: 4.2.00096
        size: 12.74M
        --------------------
        name: Atom
        path: /Volumes/munki_repo/pkgsinfo/Productivity/Atom-1.9.7.plist
        version: 1.9.7
        size: 86.89M
{% endhighlight %}

If this all looks good, you're ready to actually start removing things.

One of the nice things about Spruce is that instead of deleting, it can move
things. It can even `cp` and then `git rm` them so that you don't have to work
so hard to maintain your version controlled repo. Spruce will create the
necessary folder structure below the folder you specify so that it matches the
repo from which the deprecated items originate.

The basic deprecation, saving the three most recent of any needed item, would
look like this:
{% highlight Bash %}
spruce deprecate --auto 3
{% endhighlight %}

To move to an archive for emergency purposes:
{% highlight Bash %}
spruce deprecate --auto 3 --archive /Volumes/SomeBackupDrive/repo_archive
{% endhighlight %}

Finally, to do a git-aware deprecation:
{% highlight Bash %}
spruce deprecate --auto 3 --archive /Volumes/SomeBackupDrive/repo_archive --git
{% endhighlight %}

The deprecate interface looks a little different. This is an interactive tool,
so there is a prompt to continue, and feedback on the success or failure of
each item. Here is a more thorough example of a deprecation:
{% highlight Bash %}
spruce deprecate --auto 3 --archive /Volumes/SomeBackupDrive/repo_archive --git
Items to be removed
---------------------------------------------------------------------------
<pkginfo + pkg> AdobeAIR 21.0.0.215 (10.5.0 - None): 58.20M
---------------------------------------------------------------------------
<pkginfo + pkg> AdobeFlashPlayer 21.0.0.242 (10.5.0 - None): 17.70M
<pkginfo + pkg> AdobeFlashPlayer 22.0.0.192 (10.5.0 - None): 17.74M
---------------------------------------------------------------------------
<pkginfo + pkg> AdobeReaderDC 15.010.20060 (10.9.0 - None): 142.92M
<pkginfo + pkg> AdobeReaderDC 15.016.20039 (10.5.0 - None): 147.96M
---------------------------------------------------------------------------
<pkginfo + pkg> AnyConnect 4.2.00096 (10.5.0 - None): 12.74M
<pkginfo + pkg> AnyConnect 4.2.01035 (10.5.0 - None): 12.92M Requires: 1
---------------------------------------------------------------------------
<pkginfo + pkg> Atom 1.9.6 (10.8 - None): 86.94M
<pkginfo + pkg> Atom 1.9.7 (10.8 - None): 86.89M
---------------------------------------------------------------------------
<pkginfo + pkg> Box Sync 4.0.7693 (10.7.0 - None): 14.31M
<pkginfo + pkg> Box Sync 4.0.7697 (10.7.0 - None): 14.31M
---------------------------------------------------------------------------
<pkginfo + pkg> CitrixReceiver 12.1.0 (10.6 - None): 44.49M
# ... Clipped for blogging clarity
Items to be removed from manifests:

Are you sure you want to continue? (Y|N):
# ... Lots of stuff getting moved to the archive.
{% endhighlight %}

In the last example, there was a heading that read "Items to be removed from
manifests". When we look at some of the other deprecation features, we'll
explore how Spruce can not only remove packages and pkginfo files, but it can
also remove entries from manifests.

Next time we'll talk about some of the other reports, and how to remove items
from the repo in other ways that protect you from having to do repetitive work
and making mistakes.

Clone or download the current [Spruce on GitHub](https://github.com/sheagcraig/spruce-for-munki).
