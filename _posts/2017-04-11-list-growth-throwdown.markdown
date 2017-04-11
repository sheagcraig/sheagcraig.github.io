---
title: "Python List Growth Throwdown"
date: 2017-04-11T10:00:00-05:00
comments: true
layout: single
---
![Face kick]({{ site.baseurl }}/images/2017-04-11-list-growth-throwdown/face_kick.jpg)

### Inplace List Addition vs. List Extend

I wanted to know whether it was "better" to do

{% highlight Python %}
some_list += another_list
{% endhighlight %}

or

{% highlight Python %}
some_list(another_list)
{% endhighlight %}

So I cooked this up:

{% highlight Python %}
import timeit

results = {}

for i in xrange(100):
    add_list = "[" + ", ".join(str(x) for x in xrange(i)) + "]"
    inplace_add = timeit.timeit("l += " + add_list, setup="l = ['initial_item']", number=1000000)
    extend = timeit.timeit("l.extend({})".format(add_list), setup="l = ['initial_item']", number=1000000)
    results[i] = (inplace_add, extend)
{% endhighlight %}

### Winner-ish
The in-place addition method (theoretically called when you use the '+='
operator with lists as the left and right sides) is marginally faster. But not
enough to make me really care whether to use one vs. the other.

![Science]({{ site.baseurl }}/images/2017-04-11-list-growth-throwdown/list_growth_mega_battle.png )

### Methodology
Just for safesies I tried this with a variety of right-hand list sizes, with
1-100 elements. I then timed each one 1,000,000 times. There are some strange
bumps in the graph where the extend method beats in-place addition, but I
attribute this to my computer having to work harder for the chorus of this
sweet Mastodon song I'm listening to.
