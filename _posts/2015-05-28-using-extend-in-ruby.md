---
layout:     post
show_meta: true
title:      Using extend in Ruby
header:     Using extend in Ruby
date:       2015-05-28 22:40:00
summary:  Observations of using include and extend in Ruby
categories: ruby metaprogramming
author: Chee Yeo
---

In Ruby, it is possible to enhance the functionality of an existing object by
including a module into a class using a mixin. It is commonplace to see code like this:

{% highlight ruby %}
module DateMethods
  def self.included(base)
    base.extend ClassMethods
  end

  def calculate_interval
    (Date.today - @elapsed_created).to_i
  end

  module ClassMethods
    def interval; 3; end
  end
end

class MyClass
  include DateMethods

  attr_accessor :started
  def initialize(date:)
    @started = Date.parse(date)
  end
end
{% endhighlight %}

The example above is trivialized but hopefully helps to illustrate my observations.
We have a module called __'DateMethods'__ which we are including into __'MyClass'__.

Within the __'DateMethods'__ module, the __'self.included'__ callback allows us to call __'extend'__ on the singleton object of __'MyClass'__ in order to extend the __'ClassMethods'__ module to create class methods.

While this approach works and it is something I have been doing myself,
I can't help but feel that this is not as syntactically pleasing as calling __'MyClass.extend(DateMethods)'__ which reads better.

We are also overriding __'include'__ to create an __'extend'__ when Ruby allows us to extend a module directly.

Rewriting the example above but using __'extend'__ instead:

{% highlight ruby %}
module DateMethods
  def self.extended(base)
    base.include InstanceMethods
  end

  # class methods stay in the module body
  def interval; 3; end

  module InstanceMethods
    def calculate_interval
      (Date.today - @elapsed_created).to_i
    end
  end
end

class MyClass
  attr_accessor :started
  def initialize(date:)
    @started = Date.parse(date)
  end
end

MyClass.extend(DateMethods)
myclass = MyClass.new(date: "2015-05-28")
puts myclass.calculate_interval # => 1
puts MyClass.interval # => 3
{% endhighlight %}

Ruby provides a callback __'extended'__ which gets invoked whenever the receiver is used to extend an object. Within the __'extended'__ callback, we then __'include'__ the instance methods.

This is more flexible than __'include'__ itself as it can't be called on instances directly. In addition, __'include'__ affects the entire class itself which may
not be the intention if all you need is to have certain behavior at certain
times. By using __'extend'__ on instances, we can achieve a kind of dynamic code
loading:

{% highlight ruby %}
module Pagination
  def paginate(page = 1, items_per_page = size, total_items = size)
    @page = 1
    @items_per_page = items_per_page
    @total_items = total_items
  end

  attr_reader :page, :items_per_page, :total_items

  def total_pages
    (total_items / items_per_page).ceil
  end

end

collection = %w[first second third]

collection.extend(Pagination)
collection.paginate(1, 3, 10)
{% endhighlight %}

In the above example adapted from a [James Earl Gray article on mixins](http://graysoftinc.com/rubies-in-the-rough/learn-to-love-mix-ins){:target="_blank"}, only the __'collection'__ object has the __'Pagination'__ module methods since the __'extend'__ is on the singleton/metaclass of __'collection'__ object itself, which restricts the scope of the __'Pagination'__ behavior to be contained and not affecting the entire class. We could just have easily swapped out __'Pagination'__ with another module at runtime.

In summary, I feel this is a more flexible approach to writing more maintable, dynamic
code.

Happy hacking!









