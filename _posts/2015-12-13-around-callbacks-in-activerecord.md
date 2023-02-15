---
layout:     post
show_meta: true
title:      Using around_save callbacks in ActiveRecord
header:     Using around_save callbacks in ActiveRecord
date:       2015-12-13 00:00:00
summary:  Using the around_save callback in ActiveRecord models for further processing
categories: rails ruby ActiveRecord
author: Chee Yeo
---

It is fairly common to see 'before' and 'after' callbacks in Rails models to handle instructions for the model to execute prior to and after its state has changed.

However, there might be a need to execute some custom actions in the middle of this callback flow and you might not have control over the callback to make the changes as this might be from say an Engine or a gem.

{% highlight ruby linenos %}
class B < ActiveRecord::Base
  has_many :c
  belongs_to :a
end

class C < ActiveRecord::Base
  belongs_to :b
end


# this module could be from a gem or engine
module Audit
  def self.included(klass)
    klass.extend ClassMethods
  end

  module ClassMethods
    has_many :b

    after_save :create_b

    def create_b
      # saves a record of type B
    end
  end
end

class A < ActiveRecord::Base
  include Audit

  # how do we create C which has a reference to B ??
end

{% endhighlight %}

Take the example above. We have an ActiveRecord model A, whereby after saving it, it creates a record in the database of type B.

The callback is included into A from an Audit module which is external to the application.

How do we trigger a create on C but after the first callback so that we can obtain a reference to B?

The approach I have taken is to use an around_save. I can yield to the main class callback, in this case, creating B, and then execute my custom action to create C afterwards.

The following ruby code demonstrates the idea:
{% highlight ruby linenos %}
def test_yield
  puts "Hello World!"
  yield
  puts "Goodbye World!"
end

test_yield{ puts "Custom processing here..." } # => Hello World!; Custom processing here...; Goodbye World!
{% endhighlight %}

The test_yield method calls the block when it reaches yield and passes control to the code inside the block. Once thats done, control returns to the end of the original method.

I can use an around_save callback which calls yield on the main class to run its callbacks and then have it run my custom code after:

{% highlight ruby linenos %}
class A < ActiveRecord::Base
  include Audit

  around_save :create_c

  def create_c
    # calls the after_save defined in Audit module first
    yield

    # custom call to create C
    C.create!(b: @b)
  end
end
{% endhighlight %}

Happy Hacking!
