---
layout:     post
show_meta: true
title:      Using Ruby Refinements in Rails
header:     Using Ruby Refinements in Rails
date:       2016-06-29 00:00:00
summary: Refining NilClass over using try
categories: ruby rails
author: Chee Yeo
---

Its not very often I get the chance to use refinements in Rails. My past experiences with it in the context of Rails applications has been less than satisfactory due to the way the framework handles code reloading and the way refinements' scopes work.

However, I feel strongly enough to use refinements in a recent project due to the use / abuse of `ActiveSupport#try` method.

Supposing we have an Order class that has a single attribute, `date`. Also assuming that we have a method which calls `to_date` on that attribute like so:

{% highlight ruby linenos %}
class Order
  attr_accessor :date

  def confirmed_date
    @date.to_date
  end
end

o = Order.new
puts o.date.to_date
{% endhighlight %}

If the `date` attribute is nil, the above would throw an `undefined method on NilClass error`. A common pattern in Rails application is to wrap the above in a `try` method from ActiveSupport:

{% highlight ruby linenos %}
require 'active_support'
require 'active_support/core_ext/object/try'

class Order
  attr_accessor :date

  def confirmed_date
    @date.try(:to_date)
  end
end
{% endhighlight %}

Using the `try` method will return `nil` and not throw an exception. Underneath the hood, ActiveSupport has overridden `NilClass` with a `try` method to only return nil, disregarding any arguments passed into it:

{% highlight ruby linenos %}
# from lib/active_support/core_ext/object/try.rb

class NilClass
  def try(*args)
    nil
  end

  def try!(*args)
    nil
  end
end
{% endhighlight %}

This pattern is common in most Rails applications, and if you study the Rails source code, there are many examples given of this specific usage of `try`. But from a ruby perspective, this seems like a kind of anti-pattern to me. NilClass is just another object from the ruby ecosystem so why can't we just define the missing `to_date` method on it in this case and not have to worry about calling another method from a dependency to deal with it? My immediate thought is to apply the NullObject pattern but I think it is overkill in this use case since there is only 1 method calling `try`.

Rather than overriding NilClass, my approach is to define `to_date` on it using Ruby refinements which I can use in the context I decide is applicable like so:

{% highlight ruby linenos %}
module NilClassRefinements
  refine NilClass do
    def to_date
      nil
    end
  end
end

using NilClassRefinements

class Order
  attr_accessor :date

  def confirmed_date
    @date.to_date
  end
end

o = Order.new
puts o.date.to_date.inspect
{% endhighlight %}

The above also returns `nil` if `@date` is nil and is easier to understand with the added benefit of not overriding the core data types in Ruby.

Stay curious and keep hacking!
