---
layout: post
title:  "DEBUG MACRO"
date:   2014-07-01 14:08:36
categories: c
---

{% highlight c %}
#ifdef DEBUG
#ifndef C99
#define debug(fmt, args...) printf(fmt, ##args)
#else
#define debug(fmt, ...)	printf(fmt, __VA_ARGS__)
#endif
#else
#define debug(fmt, args...)
#endif
{% endhighlight %}
