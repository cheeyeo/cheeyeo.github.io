---
layout:     post
show_meta: true
title:      Ruby procs and collections
header:     Ruby procs and collections
date:       2016-08-28 00:00:00
summary: Using custom procs in collections
categories: ruby benchmark
author: Chee Yeo
---

In Ruby, we can use `Enumerable#map` to process collections. The `map` method takes a block:

{% highlight ruby linenos %}
names = %w(ant)

names.map{ |x| x.upcase }
{% endhighlight %}

We can also pass in custom objects which respond to `to_proc`:

{% highlight ruby linenos %}
class Double
  def to_proc
    proc{ |n| n * 2 }
  end
end

arr = [1.0, 2.0, 3.0]
arr.map(&Double.new) # => [2.0, 4.0, 6.0]
{% endhighlight %}

In Ruby 2.3+, `Hash` has a `to_proc` method which means we can pass a hash into a collection:

{% highlight ruby linenos %}
h = { foo: 1, bar: 2, baz: 3 }

[:foo, :bar].map(&h) #=> calls h.to_proc
{% endhighlight %}

I decided to do a quick benchmark on how efficient the block method of map runs compared to the using a proc:

The results show that using `proc` is slower than calling `map` with a block:

{% highlight ruby linenos %}
#/usr/bin/env ruby

require "benchmark/ips"

arr = [1.0, 2.0, 3.0]

h = { foo: 1, bar: 2, baz: 3 }

class Double
  def to_proc
    proc{ |n| n * 2 }
  end
end

Benchmark.ips do |x|
  x.report("arr.map{ |x| x*2 }"){
    arr.map{ |x| x*2 }
  }

  x.report("arr.map(&Double.new)"){
    arr.map(&Double.new)
  }

  x.report("[:foo, :bar].map"){
    [:foo, :bar].map{ |x| h[x] }
  }

  x.report("[:foo, :bar].map(&h)"){
    [:foo, :bar].map(&h)
  }

  x.compare!
end
{% endhighlight %}

{% highlight ruby %}
Warming up --------------------------------------
  arr.map{ |x| x*2 }    94.558k i/100ms
arr.map(&Double.new)    40.587k i/100ms
    [:foo, :bar].map   111.149k i/100ms
[:foo, :bar].map(&h)    71.085k i/100ms
Calculating -------------------------------------
  arr.map{ |x| x*2 }      1.740M (± 5.6%) i/s -      8.699M in   5.015954s
arr.map(&Double.new)    591.703k (± 5.8%) i/s -      2.963M in   5.024683s
    [:foo, :bar].map      1.919M (±15.7%) i/s -      9.225M in   5.049601s
[:foo, :bar].map(&h)    995.047k (±15.0%) i/s -      4.834M in   5.029552s

Comparison:
    [:foo, :bar].map:  1919075.6 i/s
  arr.map{ |x| x*2 }:  1739722.6 i/s - same-ish: difference falls within error
[:foo, :bar].map(&h):   995046.9 i/s - 1.93x  slower
arr.map(&Double.new):   591703.4 i/s - 3.24x  slower
{% endhighlight %}

From the results, I learnt that using dynamic procs may be more suited for smaller collections with `map`. For larger collections, it is more efficient to stick to the block method.

Happy Hacking and stay curious!!
