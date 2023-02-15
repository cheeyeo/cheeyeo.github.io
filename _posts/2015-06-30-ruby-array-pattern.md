---
layout:     post
show_meta: true
title:      Handling inputs with different arity in Ruby
header:     Handling inputs with different arity in Ruby
date:       2015-06-30 21:56:00
summary:  A pattern to deal with input arguments of various arity
categories: ruby pattern
author: Chee Yeo
---

In a recent Ruby project at work, I undertook a feature request which resulted in a fundamental change in one of the utility classes in the project.

The class in question takes as input, an array of file objects for processing. Due to the requirement change, this class would now have to deal with both single and multiple file objects.

My initial first attempt was clumsy to say the least:

{% highlight ruby linenos %}
class MyFiles
  def initialize(input:)
    @data = input
  end

  def process
    if @data.respond_to?(:each)
      @data.each{|f| process_file(f)}
    else
      process_file(@data)
    end
  end
end
{% endhighlight %}

My refactoring has resulted in MyFiles knowing too much about the the type of the input passed in by checking if the input responds to __:each__.

Also, this is not very DRY as subsequent code that has to process __@data__ will duplicate the __process_file__ method call and calling it either in an each block if it is an array or on its own for single inputs.

After further thinking, I realised that the only variant is the @data input which could either be a collection of files or a single file input. If we transform @data into an __Array__, then we will have a collection to work with and not have to check for the type of input. my second attempt at refactoring goes like this:

{% highlight ruby linenos %}
class MyFiles
  def initialize(input:)
    @data = Array(input)
  end

  def process
    @data.each do |f|
      process_file(f)
    end
  end
end
{% endhighlight %}

By wrapping input with __Kernel#Array__, we ensure that __@data__ is always an array and I don't have to worry about the input type anymore.

Specifically, Array does the following:

{% highlight ruby linenos %}
Array(nil) => []

Array("one") => ["one"]

Array([1,2,3]) => [1,2,3]
{% endhighlight %}

By transforming a variable input type into an array we make the class more flexible in terms of the inputs it can accept and a flexible interface to work with.

Happy Hacking!!!


