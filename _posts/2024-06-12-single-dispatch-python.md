---
layout: post
show_meta: true
title: Using single dispatch in python
header: Using single dispatch in python
date: 2024-06-12 00:00:00
summary: How to use single dispatch in python
categories: python single-dispatch
author: Chee Yeo
---

A common issue I come across when building python applications that support multiple types is to utilize an **if/else** block construct to handle issues of initializing different types of objects in a class method or to perform different actions based on a given input type.

An example could be a function to process an input object and given it's type, do it in a specific way:

{% highlight python %}
def myfunction(input) -> None:
    if isinstance(input, int):
       print('Processing an int...')
    elif isinstance(input, str):
       print('Processing a str...')
{% endhighlight %}

While the above is a contrived example, in real world usage, this is a tricky problem to debug and source of potential bugs. We need to keep updating any code which checks for the type of an object in the event of adding a new type to check for. Although we could refactor the above code, it might not always be possible if the type checking is within an external dependency such as an imported library.

A better option would be to use something like **singledispatch**. The decorator allows you to define a single generic method as a source and to register the same method which takes different types as arguments.

For instance, given the example above:

{% highlight python %}
from functools import singledispatch

@singledispatch
def myfunction(input):
    raise NotImplementedError('Not implemented in parent method!')


@myfunction.register
def _(input: int):
    print('Processing an int...')


@myfunction.register
def _(input: str):
    print('Processing a str...')
{% endhighlight %}

We import the **singledispatch** decorator from the functools module. We define the **myfunction** with the decorator as the generic function. We register copies of the generic function that takes different inputs using the decorator **@myfunction.register**. In the example above, we define it twice, one for handling integers, and another for handling strings. When we run the example, it works the same way as the original code.

Note that, the first argument to the registered function has to be the type of the target you want to invoke it with.

A real-world example could be a single function that performs different kinds of calculation based on the given input type:

{% highlight python %}
from enum import Enum

class CalcType(Enum):
    MEAN = 0
    MODE = 1
    MEDIAN = 2


def perform_calculation(kind: CalcType, path: str) -> float:
    if kind == CalcType.MEAN:
        return MeanCalculator(path)
    elif kind == CalcType.MEDIAN:
        return MedianCalculator(path)
    elif kind == CalcType.MODE(path):
        retuen ModeCalculator(path)
{% endhighlight %}

The function above performs a different kind of calculation based on the enum type specified in the input. One can see that this gets complicated as different types of calculations are added to the if/else conditional.

By refactoring it to use single dispatch, it's easier to read and reason about. Note that, singledispatch requires the first argument in the registered functions to be classes so we can't use enums. The refactored code below uses data classes to represent the same intent:

{% highlight python %}
from typing import Any
from functools import singledispatch
from dataclasses import dataclass, field
import statistics as st
import numpy as np


@dataclass
class MeanOp:
    data: list = field(default_factory=list)


@dataclass
class MedianOp:
    data: list = field(default_factory=list)


@dataclass
class ModeOp:
    data: list = field(default_factory=list)


class MeanCalculator:
    def __init__(self, data):
        self.data = data

    def calculate_average(self):
        return np.mean(self.data)
    

class ModeCalculator:
    def __init__(self, data):
        self.data = data

    def calculate_average(self):
        return float(st.mode(self.data))
    

class MedianCalculator:
    def __init__(self, data):
        self.data = data

    def calculate_average(self):
        return np.median(self.data)


@singledispatch
def perform_calculation(kind) -> float:
    raise NotImplementedError("Not implemented")


@perform_calculation.register
def _(kind: MeanOp) -> float:
    return MeanCalculator(kind.data).calculate_average()


@perform_calculation.register
def _(kind: MedianOp) -> float:
    return MedianCalculator(kind.data).calculate_average()


@perform_calculation.register
def _(kind: ModeOp) -> float:
    return ModeCalculator(kind.data).calculate_average()


if __name__ == '__main__':
    data = [1, 2, 3, 1]
    print(f'Mean: {perform_calculation(MeanOp(data))}')
    print(f'Median: {perform_calculation(MedianOp(data))}')
    print(f'Mode: {perform_calculation(ModeOp(data))}')
{% endhighlight %}

While the refactored code is easier to understand, it's also more verbose and not as DRY. This is due to the use of the dataclasses as matchers to the registered functions. The first argument of any registered function has to be a class and additional arguments will not be recognised and will raise the NotImplementedError in the generic function. By wrapping the input as a data class, we can also perform additional validation to it.

There is another variant **singledispatchmethod** which is used within classes:

{% highlight python %}
from typing import Any
from functools import singledispatchmethod
from dataclasses import dataclass, field
import statistics as st
import numpy as np


@dataclass
class MeanOp:
    data: list = field(default_factory=list)


@dataclass
class MedianOp:
    data: list = field(default_factory=list)


@dataclass
class ModeOp:
    data: list = field(default_factory=list)


class MeanCalculator:
    def __init__(self, data):
        self.data = data

    def calculate_average(self):
        return np.mean(self.data)
    

class ModeCalculator:
    def __init__(self, data):
        self.data = data

    def calculate_average(self):
        return float(st.mode(self.data))
    

class MedianCalculator:
    def __init__(self, data):
        self.data = data

    def calculate_average(self):
        return np.median(self.data)


class AverageCalculator:
    @singledispatchmethod
    @classmethod
    def perform_calculation(cls, kind):
        raise NotImplementedError("Not implemented")
    
    @perform_calculation.register
    @classmethod
    def _(cls, kind: MeanOp) -> float:
        return MeanCalculator(kind.data).calculate_average()
    
    @perform_calculation.register
    @classmethod
    def _(cls, kind: MedianOp) -> float:
        return MedianCalculator(kind.data).calculate_average()

    @perform_calculation.register
    @classmethod
    def _(cls, kind: ModeOp) -> float:
        return ModeCalculator(kind.data).calculate_average()
{% endhighlight %}

In the example above, we wrap the calculations within a class method. We register the dispatch method based on the type of the data class, which calls the corresponding type of object to perform the calculation. Note that, dispatch happens on the first non-self or non-class argument. In the example above, it tries to match the **kind** input type.

While the above does help with refactoring if/else blocks, it can also lead to repetition or even a more convoluted codebase. The single dispatch decorator can be useful for extending imported modules. For instance, if we import an existing module that has a single function we wish to generalise, we can apply the singledispatch decorator to it and register additional versions of it using register.

Other approaches for refactoring code that checks for instance types may include:

* Programming to an interface. Use typing.Protocol to create and implement protocols which can be implemented in concrete classes, leading to static duck typing.

* Use of design patterns to refactor. For example, the calculator class can be rewritten to use the Strategy design pattern.

H4PPY H4CK1NG !