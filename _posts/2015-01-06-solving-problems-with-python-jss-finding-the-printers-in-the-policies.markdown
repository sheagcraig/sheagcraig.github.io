---
author: scraig
comments: true
date: 2015-01-06 20:47:33+00:00
layout: post
slug: solving-problems-with-python-jss-finding-the-printers-in-the-policies
title: 'Solving Problems With python-jss: Finding the Printers in the Policies'
wordpress_id: 46
categories:
- python-jss
---

I wanted to start showing some short examples of easy and quick python scripts
that I have used in my day-to-day admin job. Today I got around to working on a
broken printer installation. One of our printers had a driver that was not
working, and needed to be switched out.

Adding the package to the JSS and creating an update policy was no big deal,
but I wanted to know _exactly_ which policies distributed that printer. If you
need to change a printer's PPD file, you have to remove and then re-add a
printer; and when you remove a printer from Casper Admin, that printer gets
silently dropped from all policies as well. We have a large number of policies,
and I don't like to make mistakes, so I wanted to get this right the first
time.

python-jss to the rescue!

I of course already have my python-jss preferences set up (see the
[Wiki!](https://github.com/sheagcraig/python-jss/wiki/Configuration)), so we
can start with:
    
{% highlight python %}    
    import jss
    jss_prefs = jss.JSSPrefs()
    j = jss.JSS(jss_prefs)
{% endhighlight %}

and we have a functioning JSS object.

Next, we are interested in looking at Policies, so let's grab them:
    
{% highlight python %}    
    all_policies = j.Policy().retrieve_all()
{% endhighlight %}

Notice that I skip the intermediary step of assigning the results of
`j.Policy()` to a variable. This would just give us a list of policy ID's and
names, which isn't enough information. We want the entire XML for the policies.

Speaking of which, we need to know a little bit about the structure of a policy
to be able to find the data that we're interested in (we can't just "Find")
through the whole text, since this is XML, and structure is important! So,
doing a little research, I pulled up one of my printer-installer policies and
looked at the relevant data. Here's how to do it, and what it looks like (I
trimmed out all of the unimportant stuff):

{% highlight python %}    
    >>> j.Policy(287)
    <policy>
        <general>
            <id>287</id>
            <name>Install PS Classroom  and Faculty Printers</name>
        </general>
        <printers>
            <size>2</size>
            <leave_existing_default></leave_existing_default>
            <printer>
                <id>263</id>
                <name>PSColor1</name>
                <action>install</action>
                <make_default>false</make_default>
            </printer>
            <printer>
                <id>264</id>
                <name>PSWorkroom</name>
                <action>install</action>
                <make_default>false</make_default>
            </printer>
        </printers>
    </policy>
{% endhighlight %}

So we can see that a policy has a "printers" section, with a list of
"printer"s. Each "printer" has its user-facing name in a subelement named
"name". Also, the policy has an ID and a name which I'm interested in because I
want to make sure I know how to identify them later. Fortunately, pretty much
all JSSObject subclasses have a convenience accessor for .id and .name.

Now, we just loop through all of the policies looking for a printer with the
name we're looking for; in this case "LSCopier". This is slightly challenging
because it involves using the interface to `ElementTree.Element`. Let's look at
the fully expanded version first:
    
{% highlight python %}    
    for policy in all_policies:
    	for printer in policy.findall('printers/printer'):
    		if printer.findtext('name') == 'LSCopier':
    			print('Printer found in ID %s: %s' %
    			      (policy.id, policy.name))
{% endhighlight %}

Running this short (10 lines) gets me my results. Indeed, I could probably
parameterize this and re-use it later... Which I'll do next. Also, let's trim
it up into a list comprehension.
    
{% highlight python %}    
            results = [(policy.id, policy.name) for policy in all_policies for printer in policy.findall('printers/printer') if printer.findtext('name') == search_name]
{% endhighlight %}
    
(There's no way this will fit on a single line in this theme... Sorry!) This
nets us a "results" array filled with tuples of policy names and ids that match
our argument "search_name". To pull this off we need to import sys, and make a
few more changes.

Here is the entire finished script:
    
{% highlight python %}    
    #!/usr/bin/python
    
    import sys
    
    import jss
    
    
    jssPrefs = jss.JSSPrefs()
    j = jss.JSS(jssPrefs)
    
    if len(sys.argv) > 1:
        all_policies = j.Policy().retrieve_all()
        for search_name in sys.argv[1:]:
            print "Searching for %s" % search_name
            results = [(policy.id, policy.name) for policy in all_policies for printer in policy.findall('printers/printer') if printer.findtext('name') == search_name]
            for result in results:
                print('Found in %s: %s' % result)
{% endhighlight %}

This would be easy to expand into a more general XML search utility for
auditing your Policies. For example, you may want to search for policies that
install a particular package (although
[jss_helper](https://github.com/sheagcraig/jss_helper) already does that). Or
you might want to search for policies that Recon.
