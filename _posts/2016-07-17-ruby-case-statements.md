---
layout:     post
show_meta: true
title:      Performing numeric operations in Ruby case statement
header:     Performing numeric operations in Ruby case statement
date:       2016-07-17 00:00:00
summary: Numeric operation in Ruby case statements
categories: ruby
author: Chee Yeo
---

In a recent project, I wrote a standard `case` statement which runs a conditional based on the outcome of performing a numeric operation on a value.

A sample of the code is as follows:

{% highlight ruby linenos %}
n = 2

case n
when (n % 2 ==0) then "even"
else puts "odd"
end
{% endhighlight %}

In theory, the output should be `even` given that n is divisible by 2. However, the above will output `odd` despite what the value of `n` is. This is due to the way `when` works in a case statement.

From the above example, the `when` statement is translated as:

{% highlight ruby linenos %}
(n % 2 == 0) === n # => false
{% endhighlight %}

which will always return false, hence printing `odd` no matter what variable n holds.

The only way to handle numeric operations in Ruby case statements is to encapsulate it inside a proc and use that in the `when` statement. Rewriting the above:

{% highlight ruby linenos %}
even = ->(x){ x % 2 == 0 }

n = 2

res = case n
when even "even"
else "odd"
end

res # => "even"
{% endhighlight %}

This works because underneath the hood, proc aliases the threequals `===` operator to the `call` method. It is important to realise that the `===` operator is a `method` and should not be treated as an equality operation.

This has actually caused me some confusion and led to some debugging. I hope it helps someone with a similar issue.

Stay curious and keep hacking!
