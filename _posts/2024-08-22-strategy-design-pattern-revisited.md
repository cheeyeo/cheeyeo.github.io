---
layout: post
show_meta: true
title: Revisiting the Strategy design pattern
header: Revisiting the Strategy design pattern
date: 2024-08-22 00:00:00
summary: Review of the Strategy design pattern using python typing
categories: python design-pattern strategy
author: Chee Yeo
---

[previous article on implementing the Strategy design pattern in python]: https://www.cheeyeo.dev/python/design-pattern/strategy/2023/11/21/python-design-pattern-strategy/

In a [previous article on implementing the Strategy design pattern in python], I created a fictious example using python's abstract base class inheritance as an implementation. In this post, I would like to implement a different approach using python `typing` module.

The scenario was to implement a solution for a cloud service which accepts a list of parameters to update the cloud service. Each parameter consists of a command and its inputs. For example,
{% highlight python %}
[
    ("ADD_FILE", "/data/file.txt", "10"),
    ("COPY_FILE", "/data/file.txt", "/data/file2.txt"),
]
{% endhighlight %}

Given the above list of commands, the scenario was to parse each command and invoke the corresponding command with the supplied inputs. For example, the first command would be to invoke `ADD_FILE` with `/data/file.txt` and `10` as arguments.

The previous solution used ABCs and inheritance but in doing so, we ended up with an additional context class to determine which strategy to invoke. Could we simplify it further? What if we were to convert those subclasses into functions? Could we create a protocol based on functions only?

The `typing.Protocol` type allows you to define custom protocol types as class objects with specific functions and arguments it must implement. However, we can also apply the same principle for custom functions by declaring a protocol with a single `__call__` signature:

{% highlight python %}
from typing import Protocol

class Strategy(Protocol):
    def __call__(self, *vals: str) -> str: ...
{% endhighlight %}

The definition above declares a Protocol which is a callable type by implementing the `__call__` method. Since all functions in python are callable types, we can rewrite the previous strategy classes as functions.

Next, we can declare the specific functions based on the commands above:

{% highlight python %}
def add_file(file_name: str, file_size: str) -> str:
    print('IN STRATEGY ADD_FILE')
    return 'SUCCESS'


def copy_file(source_file: str, dest_file: str) -> str:
    print('IN STRATEGY COPY_FILE')
    return 'SUCCESS'
{% endhighlight %}

The `add_file` and `copy_file` functions correspond to the example commands above. Each of this function implements the `Strategy` protocol as it accepts string arguments and return a string.

Unlike the previous example, we don't have a context class object to work with. We need a way to dynamically register each of these strategy functions when the program starts. When we parse each command, we could retrieve the actual function from this list and invoke it with its arguments.

We could create a global list that stores these strategy function types and create a custom decorator that registers them on instantiation:

{% highlight python %}
# typing to specify list of type Strategy
strategies: list[Strategy] = []

# strategy registration decorator
def register_strategy(strategy: Strategy) -> Strategy:
    strategies.append(strategy)
    return strategy


@register_strategy
def add_file(file_name: str, file_size: str) -> str:
    print('IN STRATEGY ADD_FILE')
    return 'SUCCESS'


@register_strategy
def copy_file(source_file: str, dest_file: str) -> str:
    print('IN STRATEGY COPY_FILE')
    return 'SUCCESS'
{% endhighlight %}

Note how we apply the decorator to each function we regard as a strategy. The flexibility of this approach also means that we can just remove or add the decorator to each function we want to regard as a strategy.

To invoke the actual function, we need to create a function that will parse each command and select the corresponding function from the global list:

{% highlight python %}
# the context to select and apply strategy
def apply_strategy(strategy: str, inputs: list) -> str:
    selected_strategy = [fn for fn in strategies if fn.__name__ == strategy.lower()][0]
    print(isinstance(selected_strategy, Strategy))
    return selected_strategy(*inputs)
{% endhighlight %}

The function `apply_strategy` takes in a strategy and its inputs. It tries to match the function name from the global list by comparing it to the input name. If a match is found, it passes the inputs to it.

The full code listing is as follows:

{% highlight python %}
from typing import Protocol


class Strategy(Protocol):
    def __call__(self, *vals: str) -> str: ...

# typing to specify list of type Strategy
strategies: list[Strategy] = []


# strategy registration decorator
def register_strategy(strategy: Strategy) -> Strategy:
    strategies.append(strategy)
    return strategy


@register_strategy
def add_file(file_name: str, file_size: str) -> str:
    print('IN STRATEGY ADD_FILE')
    return 'SUCCESS'


@register_strategy
def copy_file(source_file: str, dest_file: str) -> str:
    print('IN STRATEGY COPY_FILE')
    return 'SUCCESS'


# the context to select and apply strategy
def apply_strategy(strategy: str, inputs: list) -> str:
    selected_strategy = [fn for fn in strategies if fn.__name__ == strategy.lower()][0]
    return selected_strategy(*inputs)


if __name__ == "__main__":
    print(strategies)

    CMDS = [
        ("ADD_FILE", "/data/file.txt", "10"),
        ("ADD_FILE", "/data/file.txt", "10"),
        ("COPY_FILE", "/data/file.txt", "/data/file2.txt"),
        ("COPY_FILE", "/data/file.txt", "/data/file.txt"),
        ("COPY_FILE", "/data/non-exists.txt", "/data/file2.txt"),
        ("COPY_FILE", "/data/file2.txt", "/data/file3.txt"),
    ]

    for cmd in CMDS:
        try:
            algo, *params = cmd
            apply_strategy(strategy=algo, inputs=params)
        except IndexError as e:
            print(f"CMD {algo} not found!. Skip")

# Outputs:
[<function add_file at 0x7f41f2b11620>, <function copy_file at 0x7f41f2b11800>]
IN STRATEGY ADD_FILE
IN STRATEGY ADD_FILE
IN STRATEGY COPY_FILE
IN STRATEGY COPY_FILE
IN STRATEGY COPY_FILE
IN STRATEGY COPY_FILE
{% endhighlight %}

Compared to the previous approach, using protocols make the code easier to read and maintain than inheritance. 