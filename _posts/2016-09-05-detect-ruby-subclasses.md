---
layout:     post
show_meta: true
title:      Detect subclasses in Ruby
header:     Detect subclasses in Ruby
date:       2016-09-05 00:00:00
summary: Dynamically detecting subclasses in Ruby
categories: ruby metaprogramming objectspace
author: Chee Yeo
---

I recently came across a problem which involved being able to select subclasses dynamically based on user input.

Given a master class of say `A` and a user input string, we should be able to instantiate a new subclass of `A`.

If we are using Rails framework, we can use `ActiveSupport#CoreExt` which has monkey-patched `Class` with two additional methods: `descendants` and `subclasses`

The source code from ActiveSupport is as follows:

{% highlight ruby linenos %}
class Class
  def descendants
    descendants = []
    ObjectSpace.each_object(singleton_class) do |k|
      # prepends to front of descendants
      descendants.unshift k unless k == self
    end

    descendants
  end

  def subclasses
    subclasses, chain = [], descendants
    chain.each do |k|
      subclasses << k unless chain.any? { |c| c > k }
    end
    subclasses
  end
end
{% endhighlight %}

Given the following class hierarchy we can use the extensions like so:

{% highlight ruby linenos %}
require 'active_support'
require 'active_support/core_ext'

class A; end
class B < A; end
class C < B; end

puts A.descendants # => [B,C]

puts A.subclasses # => [B]
{% endhighlight %}

Calling `ObjectSpace.each_object` with a class or module argument returns a list of matching classes or modules including its subclasses.

In this case, we are passing in `ObjectSpace.each_object(A.singleton_class)` which returns matching classes whose singleton class is either `A.singleton_class` or a subclass of it:

{% highlight ruby linenos %}
ObjectSpace.each_object(A.singleton_class).to_a # => [A,B,C]

B.singleton_class.superclass # => # <Class:A>

C.singleton_class.superclass # => # <Class:B>

C.singleton_class.superclass.superclass # => # <Class:A>
{% endhighlight %}

Calling `A.descendants` returns only `[B,C]` as within the `each_object` loop it excludes `A` itself.

Calling `A.subclasses` will return only direct subclasses, namely `[B]`. This is because it loops through the descendants list and makes a class comparison whereby only the top level subclasses are selected:

{% highlight ruby linenos %}
B > C # => true

B > B # => false

C > C # => false

C > B # => false
{% endhighlight %}

`ObjectSpace` module is certainly a powerful introspection tool to have in your toolbelt if you need detailed analytics about the objects in your current ruby process.

Stay curious and keep hacking!

### Resources
- [ActiveSupport docs](http://guides.rubyonrails.org/active_support_core_extensions.html#subclasses-descendants)
