---
layout:     post
show_meta: true
title:      Ruby type coercions
header:     Ruby type coercions
date:       2016-08-30 00:00:00
summary: How to convert between types in Ruby
categories: ruby
author: Chee Yeo
---

When developing in Ruby, we may have to convert our own custom objects into one of the built-in system types.

Ruby has built-in support for conversion protocols. By that, it means Ruby allows an object of a class to have itself converted to an object of another class.

Ruby supports three main modes of type conversions: `lenient`, `strict`, `numeric`. We will discuss the first two forms of conversions in this post.

When you call `to_s` or `to_i` on a receiver, Ruby looks for an equivalent representation
in the receiver. For instance, calling `to_s` on any class which does not implement a custom `to_s` method will return just the class name followed by the object ID as a string.

Sometimes, `strict` conversions are necessary in order to use our custom objects interchangeably with the built-in system types - we need our custom objects to work together with the core system types.

This is when the `strict conversion` functions from the standard library comes in handy.

Below is a list of these predefined system methods and their corresponding conversions:

{% highlight ruby %}
to_a      ->  Array

to_ary    ->  Array

to_enum   ->  Enumerator

to_hash   ->  Hash

to_int    ->  Integer

to_io     ->  IO

to_open   ->  IO

to_path   ->  String

to_proc   ->  Proc

to_regexp ->  Regexp

to_str    ->  String

to_sym    ->  Symbol
{% endhighlight %}


Some of the Ruby built-in classes such as Array, Hash, Regexp, String support the `try_convert` method call, which tries to transform the object passed as its argument into the corresponding type.

When `try_convert` is invoked on an object, the type it is to be converted to looks for the associated method call in the list above in order to determine how the conversion is to occur.

Below is an example of transforming a custom class of type Alphabet into an array by implementing `to_ary`:

{% highlight ruby linenos %}
class Alphabet
  def to_a
    ('a'..'z').to_a
  end
end

res = Array.try_convert(Alphabet.new) # => calls to_ary on class A
puts res.class # => Array
puts res # => ['a', 'b', 'c',...,'z']
{% endhighlight %}

This allows the use of the Alphabet class interchangeably when an array is expected.

However, we are not limited to just a single implementation.

Taking the example above, if we need to extend the conversions to the other classes, we can also add the other corresponding methods:

{% highlight ruby linenos %}
class Alphabet
  def alpha_range
    ('a'..'z')
  end

  # returns array of alphabets
  def to_ary
    alpha_range.to_a
  end

  # returns hash of alphabet as key and its position as value
  def to_hash
    ('a'..'z').each_with_object({}){ |alpha, hsh|
      hsh[alpha] = alpha_range.find_index(alpha)
    }
  end

  # returns string of all alphabets concatenated
  def to_str
    self.to_ary.join(' ')
  end

  # calls to_str to return a Regexp
  def to_regexp
    Regexp.new(self)
  end

  # returns number of alphabets in range
  def to_int
    alpha_range.count
  end
end

alphabets = Alphabet.new

Array.try_convert(alphabets) # => ['a', 'b', 'c',...,'z']

Hash.try_convert(alphabets) # => {"a"=>0, "b"=>1, "c"=>2, ..., "z" => 25}

String.try_convert(alphabets) # => 'a b c ... z'

Regexp.try_convert(alphabets) # => /a b c .... z/

Integer(alphabets)            # => 26
{% endhighlight %}

Keep hacking and stay curious!
