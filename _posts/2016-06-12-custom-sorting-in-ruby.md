---
layout:     post
show_meta: true
title:      Custom sorting in Ruby
header:     Custom sorting in Ruby
date:       2016-06-12 10:00:00
summary: Sorting with custom attributes in Ruby
categories: ruby sort custom
author: Chee Yeo
---

Assume we have a `Severity` class which represents the level of severity which may arise in a system.

This class has a single attribute: a severity string. The levels of severities can range from `"info, low, medium, high"`. If we try to sort a collection of severities in ascending order by its level, we get the following result:

{% highlight ruby linenos %}
["info", "low", "high", "medium"]
{% endhighlight %}

Likewise for a sort in DESC order based on its levels:

{% highlight ruby linenos %}
["medium", "low", "high", "info"]
{% endhighlight %}

The issue here is that the sort order does not represent the inherent meaning of the levels themselves. We want the `"high"` level to be at the front of the list when sorting in descending order and `"low"` to be at the front in ascending order. Ruby is sorting lexically and does not capture the inherent meaning of the levels.

We can develop a custom comparator class whereby we implement the sort order ourselves.

{% highlight ruby linenos %}
class Severity
  attr_accessor :level

  def initialize(level)
    @level = level
  end
end

module SeverityComparator
  LEVELS = ["info", "low", "medium", "high"]

  module_function
  def compare(a, b)
    LEVELS.index(a.level) <=> LEVELS.index(b.level)
  end
end
{% endhighlight %}

The SeverityComparator module holds an array `LABELS` whereby we predefine the severity levels in the order we want.

Within the `compare` method, we compare the position of the object's index within the `LEVELS` array based on its level attribute. For example:

{% highlight ruby linenos %}
arr = ["info", "low", "medium", "high"].each_with_object([]){|a, memo|
  memo << Severity.new(a)
}

arr.sort{|a,b| SeverityComparator.compare(a,b)}.map(&:level)
# => ["info", "low", "medium", "high"]

arr.sort{|a,b| -SeverityComparator.compare(a,b)}.amp(&:level)
#=> ["high", "medium", "low", "info"]
{% endhighlight %}

Supposing that we also discover that the severity levels can also range from `"1.0" to "5.0"`. The flexibility of this approach means that we can just add new levels to the `LEVELS` array and still use the same comparator:

{% highlight ruby linenos %}
module SeverityComparator
  # adding new levels
  LEVELS = ["1.0", "2.0", "3.0", "4.0", "5.0", "info", "low", "medium", "high"]

  # .....
end

arr2 = ["1.0", "2.0", "3.0", "4.0", "5.0"].each_with_object([]){|a, memo|
  memo << Severity.new(a)
}

arr2.sort{|a,b| SeverityComparator.compare(a,b)}.map(&:level)
# => ["1.0", "2.0", "3.0", "4.0", "5.0"]

arr2.sort{|a,b| -SeverityComparator.compare(a,b)}.map(&:level)
# => ["5.0", "4.0", "3.0", "2.0", "1.0"]

# sorting still works as before
arr.sort{|a,b| SeverityComparator.compare(a,b)}.map(&:level)
# => ["info", "low", "medium", "high"]
{% endhighlight %}

Keep Hacking!!!

## Further References

- [Custom sorting example](http://sandmoose.com/post/59750873517/sorting-with-a-custom-alphabet-in-ruby){:target="_blank"}
- [Sort vs Sort-by](http://brandon.dimcheff.com/2009/11/18/rubys-sort-vs-sort-by.html){:target="_blank"}
