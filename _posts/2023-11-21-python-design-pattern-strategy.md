---
layout: post
show_meta: true
title: Using the Strategy design pattern
header: Using the Strategy design pattern
date: 2023-11-21 00:00:00
summary: Using the Strategy design pattern in refactoring python code
categories: python design-pattern strategy
author: Chee Yeo
---

[Strategy Pattern]: https://refactoring.guru/design-patterns/strategy

In a recent practice of code exercises from CodeSignal, I came upon an interesting problem of designing up a fictious cloud storage service which is able to accept a list of commands and its parameters to update the storage service.

The commands are provided in a list for example:
{% highlight python %}
[
    ("ADD_FILE", "/data/file.txt", "10"),
    ("COPY_FILE", "/data/file.txt", "/data/file2.txt"),
]
{% endhighlight %}

The commands are to be parsed sequentially. Each command either updates the storage state or returns an error message of why its not successful. The returned responses are in the form of boolean strings i.e. 'true' or 'false'.

Given the above sequence, a file of file size '10' is to be added to the storage. Next, a copy of it is to be made and stored into storage. The final storage state and reults become:

{% highlight python %}
# storage state
{'/data/file.txt': '10', '/data/file2.txt': '10'}

# responses
{'true', 'true'}
{% endhighlight %}

My initial attempt was to use a single function with an if-else loop. It iterates over the command lists and parses each command into its name and parameters, which are delegated to additional functions to perform the operation with its input parameters. While it works in the first stages, the code became unwiedly by the second stage as more actions are added to the command list. 

I started to research into using design patterns to refactor the code and came across the **Strategy Pattern**. The strategy pattern allows you to define a set of algorithms in its own bespoke classes and allow them to be interchageable at runtime. This is made possible through the use of a context object which defines the algorithm or strategy to delegate to at runtime.

We define a base class of **Strategy** which the algorithms can inherit from. There is only a single function of **do_work** which accepts the input parameters:

{% highlight python %}
from abc import abc


class Strategy(ABC):
    """
    Abstract base class for all algos
    """

    def do_work(self, data: list[Any]) -> str:
        raise RuntimeError('Must be implemented in child classes')

{% endhighlight %}

Next, we define the individual algorithms or strategies:

{% highlight python %}
class AddFile(Strategy):
    """
    Adds file to storage
    """

    def do_work(self, data: list[Any]) -> str:
        fname, fsize = data

        if not fname in STORAGE.keys():
            STORAGE[fname] = fsize
            return 'true'
        else:
            print(f'AddFile: {fname} already exists')
            return 'false'
    

class CopyFile(Strategy):
    """
    Copies file from source to target
    """

    def do_work(self, data: list[Any]) -> str:
        source, dest = data


        if source not in STORAGE.keys():
            print(f'CopyFile: {source} not exists')
            return 'false'


        if dest == source:
            print(f'{dest} is the same as {source}')
            return 'false'
        

        STORAGE[dest] = STORAGE[source]
        print(f'{source} copied to {dest}')
        return 'true'
{% endhighlight %}

We define two subclass of `AddFile` and `CopyFile` as per the example inputs. Each class overrides the `do_work` abstract function to define its own functionality. For `AddFile`, we check if the file exists in storage. If so, we return 'false' with an error message. For `CopyFile`, we check that the source and destination filenames are not the same and that the source file exists beforehand.

Next, we define the context class which has a reference to the strategy object to use at runtime:

{% highlight python %}
class Context:
    def __init__(self, strategy: Strategy | None = None) -> None:
        self._strategy = strategy
    

    @property
    def strategy(self) -> Strategy:
        return self._strategy
    

    @strategy.setter
    def strategy(self, strategy: Strategy) -> None:
        self._strategy = strategy


    def do_work(self, inputs: list[Any]) -> str:
        result = self._strategy.do_work(inputs)
        return result

{% endhighlight %}


The `Context` class takes in an initial strategy object or None. We define the same `do_work` function which delegates the actual work to the underlying strategy object reference and returns its results.

The complete refactored code is as follows:
{% highlight python %}
### Example of using Strategy pattern to process list of commands to process and store files into a cloud storage


from abc import ABC
from typing import Any


# Dict to emulate a storage mechanism of some kind
STORAGE = {}


class Strategy(ABC):
    """
    Abstract base class for all algos
    """

    def do_work(self, data: list[Any]) -> str:
        raise RuntimeError('Must be implemented in child classes')


class AddFile(Strategy):
    """
    Adds file to storage
    """

    def do_work(self, data: list[Any]) -> str:
        fname, fsize = data

        if not fname in STORAGE.keys():
            STORAGE[fname] = fsize
            return 'true'
        else:
            print(f'AddFile: {fname} already exists')
            return 'false'
    

class CopyFile(Strategy):
    """
    Copies file from source to target
    """

    def do_work(self, data: list[Any]) -> str:
        source, dest = data


        if source not in STORAGE.keys():
            print(f'CopyFile: {source} not exists')
            return 'false'


        if dest == source:
            print(f'{dest} is the same as {source}')
            return 'false'
        

        STORAGE[dest] = STORAGE[source]
        print(f'{source} copied to {dest}')
        return 'true'


class Context:
    def __init__(self, strategy: Strategy | None = None) -> None:
        self._strategy = strategy
    

    @property
    def strategy(self) -> Strategy:
        return self._strategy
    

    @strategy.setter
    def strategy(self, strategy: Strategy) -> None:
        self._strategy = strategy


    def do_work(self, inputs: list[Any]) -> str:
        result = self._strategy.do_work(inputs)
        return result
    


if __name__ == '__main__':
    results = []

    cmds = [
        ("ADD_FILE", "/data/file.txt", "10"),
        ("ADD_FILE", "/data/file.txt", "10"),
        ("COPY_FILE", "/data/file.txt", "/data/file2.txt"),
        ("COPY_FILE", "/data/file.txt", "/data/file.txt"),
        ("COPY_FILE", "/data/non-exists.txt", "/data/file2.txt"),
        ("COPY_FILE", "/data/file2.txt", "/data/file3.txt"),
    ]

    context = Context()

    for cmd in cmds:
        algo, *params = cmd
        
        if algo == "ADD_FILE":
            strategy = AddFile()
            context.strategy = strategy
            results.append(context.do_work(params))
        elif algo == "COPY_FILE":
            strategy = CopyFile()
            context.strategy = strategy
            results.append(context.do_work(params))
    
    print(results)
    print(STORAGE)
    assert results == ['true', 'false', 'true', 'false', 'false', 'true']
    assert len(STORAGE) == 3
{% endhighlight %}

The following link provides a more detailed explanation of the [Strategy pattern].

H4ppy H4ck1ng!