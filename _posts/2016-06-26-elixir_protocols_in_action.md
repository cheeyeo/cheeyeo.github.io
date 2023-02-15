---
layout:     post
show_meta: true
title:      Elixir protocols in action
header:     Elixir protocols in action
date:       2016-06-26 00:00:00
summary: Using Collectable protocol in Elixir
categories: elixir protocols
author: Chee Yeo
---

In a previous post, I discussed the use of Protocols in Elixir. In effect, I was struggling for an internal definition for myself as to the difference between a behaviour and a protocol, until I came across an article by Gregory Brown which has this quote from 'Growing Object Oriented Software':

> An interface defines whether two things can fit together, a protocol defines whether two things can work together.

To demonstrate how a protocol allows a custom struct to work with the Enum module, I've devised a contrived example whereby I'm implementing the `Collectable protocol` for a custom data struct.

The Enum module supports a function `into` which takes an enumerable, and a data structure and transposes the original enumerable into the given data structure and returns it.

An example of putting items from one list into another after transformation using `Enum.into/3`:

{% highlight elixir linenos %}
Enum.into([1,2,3], [], fn(x) -> x + 1 end) # => [2,3,4]
{% endhighlight %}

The second argument for into is our `Collectable` as specified in the docs. We just need to implement the `Collectable protocol` for our data struct by implementing one method called `into` and passing it the original data structure.

Assuming we have a struct called `MyCollectable`, the implementation and function declaration looks like this:

{% highlight elixir linenos %}
# this is the custom data struct we are storing the data into
defmodule MyCollectable do
  defstruct results: []
end


defimpl Collectable, for: MyCollectable do
  def into(original) do
    {original, fn
      source, {:cont, nil} -> source
      source, {:cont, value} ->
        %MyCollectable{source | results: [value|source.results]}
      source, :done ->
        %MyCollectable{source | results: :lists.reverse(source.results)}
      _, :halt -> :ok
    end}
  end
end
{% endhighlight %}

The `Collectable.into` function returns a tuple of the original collection and a function which is applied as it cycles through each item.

For `{:cont, nil}` no value is matched so it returns the original collection.

For `{:cont, value}` a value is found so it is appended to the head of the collection results list in our example but can be customized to suit.

When `{:done}` is read, it means it has reached the end of the iteration. In our case, we return an updated `MyCollectable` struct with the results reversed in the right sequence.

Finally, `{:halt}` is implemented to return `:ok` and it is to handle situations where processing is interuppted.

To use it, we can pass it into `Enum.into` like so:


{% highlight elixir linenos %}
Enum.into([1,2,3], %MyCollectable{}) # => %MyCollectable{results: [1,2,3]}

Enum.into([1,2,3], %MyCollectable{}, fn x -> x + 1 end ) # => %MyCollectable{results: [2,3,4]}
{% endhighlight %}

A [sample github repository]{:target="_blank"} for this post is available.

Keep hacking and stay curious!!

### Further information
- [Collectable documentation](http://elixir-lang.org/docs/stable/elixir/Collectable.html){:target="_blank"}

- [Enum into/2](http://elixir-lang.org/docs/stable/elixir/Enum.html#into/2){:target="_blank"}

- [Enum into/3](http://elixir-lang.org/docs/stable/elixir/Enum.html#into/3){:target="_blank"}

[sample github repository]: https://github.com/cheeyeo/Collectable_Example
