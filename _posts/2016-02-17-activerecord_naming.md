---
layout:     post
show_meta: true
title:      Getting names of ActiveRecord classes
header:     Getting names of ActiveRecord classes
date:       2016-02-17 21:00:00
summary:  Using ActiveModel::Naming to retrieve more information on names of ActiveRecord models.
categories: rails activemodel activerecord
author: Chee Yeo
---

In a recent Rails application I am working on I stumbled upon the problem of having to dynamically instantiate a particular processing class based on the type of object being passed into a particular method.

My first attempt involved getting the instance's class name using the class's name:

{% highlight ruby linenos %}

post = Post.find(1)

post.class.name # => returns "Posts"
{% endhighlight %}

However this is redundant since __ActiveRecord::Base__ by default extends __ActiveModel::Naming__:

{% highlight ruby linenos %}
# activerecord/lib/activerecord/base.rb

module ActiveRecord
  class Base
    extend ActiveModel::Naming

    ...
  end
end
{% endhighlight %}

Rather than call __class.name__, if you call __model_name__ on an ActiveRecord instance or its class, you get an __ActiveModel::Naming__ object back which contains useful naming metadata:

{% highlight ruby linenos %}
post = Post.find(1)

post.model_name

#<ActiveModel::Name:0x007fe2ea756cf8 @name="Post", @klass=Post(id: integer, title: string, body: text, created_at: datetime, updated_at: datetime), @singular="post", @plural="posts", @element="post", @human="Post", @collection="posts", @param_key="post", @i18n_key=:post, @route_key="posts", @singular_route_key="post">
{% endhighlight %}

From the output above, I was able to just pass the instance's __model_name__ object into the method and since this object has a predictable api I was able to remove type checking completely:

{% highlight ruby linenos %}
post = Post.find(1)


dynamic_processor(post)

....

def dynamic_processor(obj, opts={})
  # previous type checking code which includes calls like respond_to? etc
  # this can now be replaced by a simple hash map of class names to classes
  processors = {
    "Post" => Post,
    "Report" => Report
  }

  klass = processors[obj.model_name.name]
  klass.new(opts)
end

{% endhighlight %}

After all this time, I am still amazed by Rails and the pleasant surprises it brings to the table.

You can [check the following docs for more information on ActiveModel::Naming](http://api.rubyonrails.org/classes/ActiveModel/Naming.html){:target="_blank"}

Happy Hacking!
