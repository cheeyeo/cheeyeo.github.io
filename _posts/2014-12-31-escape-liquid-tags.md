---
layout:     post
show_meta: true
title:      Escaping Liquid Tags in Jekyllrb
header:     Escaping Liquid Tags in Jekyllrb
date:       2014-12-31 11:41:00
summary:  Escaping Liquid Tags in Jekyllrb
categories: jekyllrb liquid
author: Chee Yeo
---

[Liquid](https://github.com/Shopify/liquid/){:target="_blank"} is a template engine used in Jekyllrb.

In order to use liquid tags within code blocks without it being parsed, we need
to escape it first.

There is a tag built into liquid called ['raw'](https://github.com/Shopify/liquid/blob/master/lib/liquid/tags/raw.rb){:target="_blank"} which
will do the escaping automatically for you.

Below is an example of using the __raw__ tag to escape some liquid markup
in a pre code block:

{% highlight html linenos %}
{% raw %}
<p>{% you should be able to see the curly braces! %}</p>
{% endraw %}
{% endhighlight %}

Happy Hacking!
