---
layout: page
title: My Projects
permalink: /projects/
---

<p>From time to time the need for a side project arises - here's where I'll leaving any potentially useful ones.</p>

<h2><a href="http://configmapper.org">ConfigMapper</a></h2>

<p>ConfigMapper is a tiny package which maps application config to strongly-typed objects with no magic strings. It enables this:</p>

{% highlight c#  %}
    int myNumber = Convert.ToInt32(ConfigurationManager.AppSettings["MyNumber"]); 
{% endhighlight %}

<p>to become this:</p>

{% highlight c#  %}
    int myNumber = Configuration.AppSettings.MyNumber; 
{% endhighlight %}

<h3>What's the difference?</h3>

<p>ConfigMapper performs the type conversion automatically and doesn't use magic strings - find usages easily and rename without a fuss.</p>

<p>See the website at <a href="http://configmapper.org">http://configmapper.org</a> for more info, or head on over to <a href="http://github.com/martinburrows/configmapper">GitHub</a> for the source.</p>

<hr>


<h2><a href="http://niceguid.com">NiceGUID</a></h2>

<p>Generator of nice, human-readable GUIDs. A while back I needed to generate several GUIDs and then be able to easily recognise them later on. After generating a few with <a href="https://en.wikipedia.org/wiki/Leet">leetspeak</a>, NiceGUID was born.</p>

<a href="http://niceguid.com"><img src="/images/pages/niceguid.png" /></a>

<p>Try it for yourself at <a href="http://niceguid.com/">http://niceguid.com/</a>