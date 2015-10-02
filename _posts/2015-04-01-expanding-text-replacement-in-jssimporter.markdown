---
author: scraig
comments: true
date: 2015-04-01 18:54:02+00:00
layout: post
slug: expanding-text-replacement-in-jssimporter
title: Expanding Text Replacement in JSSImporter
wordpress_id: 190
---

After some PR's and ideas tossed around with other admins using JSSImporter, it
became pretty clear that there are a lot of creative uses of JSSImporter that
the original code doesn't make very easy, or possible, without some hacking.

One big change that comes with JSSImporter 0.3.8 is that it allows you to use
any string-type AutoPkg environment variable in your template text
substitutions.

To use a variable for text substitution, just wrap it in %'s in the template
file. For example, if I wanted to include the value of AUTOPKG_VERSION as part
of a description in my policy template, I could do this (only the relevant
parts of the XML are included for brevity!)

{% highlight xml %}    
    <description>Built with AutoPkg version %AUTOPKG_VERSION% for your pleasure!</description>
{% endhighlight %}    

One thing that many people don't realize is that AutoPkg doesn't freak out if
you add arguments or input variables to your recipe that don't "exist". You can
use this to fill arguments to the JSSImporter as well as templates; you don't
necessarily need an argument for every conceivable setting. For example, say I
want to have a second smart group scoped in my policy, but I want the name to
be overrideable with input variables.

First I would add a new input variable named something appropriately:
 
{% highlight xml %}    
    <description>Built with AutoPkg version %AUTOPKG_VERSION% for your pleasure!</description>
    <key>Input</key>
    <dict>
        <key>SecondGroupName</key>
        <string>SuperTesters</string>
    ...
{% endhighlight %}    

Then, later, in the JSSImporter arguments, in the groups array, I could have my
second group use it:
 
{% highlight xml %}    
    <key>groups</key>
    <array>
        <dict>
            <key>name</key>
            <string>%GROUP_NAME%</string>
            <key>smart</key>
            <true></true>
            <key>template_path</key>
            <string>%GROUP_TEMPLATE%</string>
        </dict>
        <dict>
            <key>name</key>
            <string>%SecondGroupName%</string>
            <key>smart</key>
            <true></true>
            <key>template_path</key>
            <string>%GROUP_TEMPLATE2%</string>
        </dict>
    </array>
{% endhighlight %}    

_and_ I can include that somewhere in a template as well. So my policy template
could include this:
 
{% highlight xml %}    
    <policy>
        <general>
            <name>Install Latest %PROD_NAME%</name>
            <enabled>true</enabled>
            <frequency>Ongoing</frequency>
            <category>
                <name>%POLICY_CATEGORY%</name>
            </category>
        </general>
        <scope> 
            
        </scope>
        <package_configuration>
            
        </package_configuration>
        <scripts>
            
        </scripts>
        <self_service>
            <use_for_self_service>true</use_for_self_service>
            <install_button_text>Install %VERSION%</install_button_text>
            <self_service_description>%SELF_SERVICE_DESCRIPTION%</self_service_description>
        </self_service>
        <maintenance>
            <recon>true</recon>
        </maintenance>
        <user_interaction>
            <message_start>Greetings %SecondGroupName%</message_start>
            <allow_users_to_defer>false</allow_users_to_defer>
            <allow_deferral_until_utc></allow_deferral_until_utc>
            <message_finish></message_finish>
        </user_interaction>
    </policy>
{% endhighlight %}    

Granted, that's kind of a contrived use, but it points to a way to manage a lot
of settings with a pretty flexible ability to later override things.

Which reminds me: Try to set all of the arguments to JSSImporter in your
recipes with input variables. This makes it a lot easier for other people to
then borrow your recipe and make it work in their environment. Indeed, once we
get it off the ground, this will be a style requirement for the "official" JSS
recipes repo... Stay tuned.
