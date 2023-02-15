---
layout:     post
show_meta: true
title:      Elixir Protocols
header:     Elixir Protocols
date:       2016-06-11 10:00:00
summary: What are protocols in Elixir and how are they useful
categories: elixir protocols
author: Chee Yeo
---

Protocols are a powerful language feature in Elixir. It allows you to specify an api which should be defined by its implementation. If you ever need to create functions that accept polymorphic types, protocols allow you to do so in a managed and organized manner.

Although Elixir Behaviours are similar, Behaviours are defined internally within each module. Protocol implementations can occur externally outside of the module. This allows for extending a module's functionality you don't have access to.

Suppose we created an Odd protocol which returns true or false depending on the type.

{% highlight elixir linenos %}
defprotocol Odd do
  @doc "Returns true if the data is considered odd"
  @fallback_to_any true

  def odd?(data)
end
{% endhighlight %}

`@doc` describes what the protocol does. The function to be implemented is `def odd?(data)`. `@fallback_to_any` is an attribute which specifies that a default implementation of `Any` is to be used in the event that no specific implementation of Odd is found for that particular type.

To implement the protocol:
{% highlight elixir linenos %}
defimpl Odd, for: Integer do
  require Integer

  def odd?(num) do
    Integer.is_odd(num)
  end
end

defimpl Odd, for: Float do
  def odd?(num) do
    Odd.odd?(trunc(num))
  end
end

defimpl Odd, for: List do
  def odd?(data) do
    Odd.odd?(Enum.count(data))
  end
end

defimpl Odd, for: Any do
  def odd?(_), do: false
end
{% endhighlight %}

`defimpl` defines the Odd protocol for the type specified in `for` in this case for both Integer and Float types.

We define an integer as odd if `Integer.is_odd` macro returns true although we can also write our own implementation here.

For a float, we define it as odd if the integer part of the float is odd. Since `trunc` returns an integer, we can pass it to the earlier implementation of odd for Integer through `Odd.odd?`

For lists, it is odd if it has an odd number of elements.

The final implementation is a default fallback for any data type for which there is no odd protocol implementation. This is required by the `@fallback_to_any` attribute else it will not compile.

Now we can call it like so:

{% highlight elixir linenos %}
Odd.odd?(1) # true

Odd.odd?(2) # false

Odd.odd?(1.9) # true

Odd.odd?(2.1) # false

Odd.odd?([1]) # true
{% endhighlight %}

To test the implementation directly:

{% highlight elixir linenos %}
Odd.Integer.odd?(1) # true

Odd.Float.odd?(1.9) # true
{% endhighlight %}

For data types which don't implement Odd it will automatically trigger the `Any` implementation of protocol which always returns false:

{% highlight elixir linenos %}
Odd.odd?(%{}) # false

Odd.odd?(:atom) # false

Odd.impl_for(%{}) # Odd.Any
{% endhighlight %}

We can also define protocols for user defined data types such as structs.

{% highlight elixir linenos %}
# assuming we created an Animal struct

defmodule Animal do
  defstruct [:hairy]
end

defimpl Odd, for: Animal do
  def odd?(%Animal{hairy: true}), do: true
  def odd?(_), do: false
end
{% endhighlight %}

Here, we define an Animal to be odd if it is hairy.

{% highlight elixir linenos %}
Odd.odd?(%Animal{hairy: true}) # true

Odd.odd?(%Animal{hairy: false}) # false
{% endhighlight %}

There are 2 useful introspection functions for protocols which I find useful:

* `__protocol__(:functions)`

   List all functions and their arity as defined by the protocol

* `impl_for(structure)`

   Returns module which implements protocol for that structure.

Example usage:

{% highlight elixir linenos %}
Odd.__protocol__(:functions) #=> [:odd]

Odd.impl_for(%Animal{}) #=> Odd.Animal

Odd.impl_for(1) #=> Odd.Integer
{% endhighlight %}

### Redefining an existing module behaviour

Lets assume we want to redefine the print format of Animal struct. `inspect` calls `Inspect.inspect` and `Inspect` is a protocol. We can redefine it like so:

{% highlight elixir linenos %}
defimpl Inspect, for: Animal do
  def inspect(animal, _opts) do
    # convert animal struct to map to get the attributes:
    attr_str = Map.delete(animal, :__struct__)
               |> Enum.reduce("", fn({k,v}, acc) -> acc <> "* #{k} -> #{v}" end)

    """
    Animal has the following attrs:

    #{attr_str}
    """
  end
end
{% endhighlight %}

Now when we call `inspect` on an Animal struct, we get the following:

{% highlight elixir linenos %}
IO.inspect %Animal{hairy: false}

# => Animal has the following attrs:
# * hairy -> false

# to ensure IO.inspect is calling Inspect.Animal
Inspect.impl_for(%Animal{}) # => Inspect.Animal
{% endhighlight %}

### Summary

Protocols are a powerful feature to help support polymorphism in Elixir. We have seen how to define our custom protocol for standard system types as well as for our own structs. We have also seen a trivial example of how to redefine a system built protocol for our own ends.

A [sample github repository]{:target="_blank"} for this post is available.

Keep hacking and stay curious!!


### Further information

- [Elixir Getting Started guide on protocols](http://elixir-lang.org/getting-started/protocols.html){:target="_blank"}
- [defprotocol documentation](http://elixir-lang.org/docs/stable/elixir/Kernel.html#defprotocol/){:target="_blank"}
- [Functions for working with Protocol](http://elixir-lang.org/docs/stable/elixir/Protocol.html){:target="_blank"}

[sample github repository]: https://github.com/cheeyeo/Elixir-Protocol
