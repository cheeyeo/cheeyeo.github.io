---
layout: post
show_meta: true
title: Intro to python protocol
header: Intro to python protocol
date: 2024-08-15 00:00:00
summary: Short introduction to python protocol
categories: python protocol
author: Chee Yeo
---

Python uses dynamic duck typing by default since the bytecode is interpreted on-the-fly by the python virtual machine. 

Later versions of python since 3.8 provides a mechanism known as a **Protocol**. A protocol specifies the methods and attributes a class must implement in order to be considered a given type. Implementing a protocol is also known as static duck typing.

There are 2 main ways python determines at runtime the type of an object:

* Nominal subtyping is based on inheritance. A class which inherits from a parent class is a subtype of its parent.

* Structural subtyping is based on the internal structure of classes. A class that implements the methods and attributes of a protocol is a type of the protocol.

The idea is similar to the use of interfaces in go-lang. When we implement all the methods defined by an interface in go-lang, we can use the custom implementation interchangeably where the interface is defined.

Suppose we have an e-commerce platform whereby we want to define and be able to apply different promotions upon cart checkout. We could create a protocol called Promotion that's callable as a function:

{% highlight python %}
from typing import Protocol

class Promotion(Protocol):
    def __call__(self, order: Order) -> Decimal: ...
{% endhighlight %}

From above, we defined the protocol to be a callable via the __call__ method, which accepts as input an Order and returns a Decimal. Any function that implements this function signature will be considered via structural subtyping to be a type of Promotion.

We can define our promotions as follows:

{% highlight python %}
promos: list[Promotion] = []


def best_promo(order: Order) -> Decimal:
    return max(promo(order) for promo in promos)


class Fidelity:
    def __call__(self, order: Order) -> Decimal:
        if order.customer.fidelity >= 1000:
            return order.total() * Decimal('0.05')
        
        return Decimal(0)


class BulkItem:
    def __call__(self, order: Order) -> Decimal:
        discount = Decimal(0)
        for item in order.cart:
            if item.quantity >= 20:
                discount += item.total() * Decimal('0.1')
        
        return discount


class LargeOrder:
    def __call__(self, order: Order) -> Decimal:
        distinct_items = {item.product for item in order.cart}
        if len(distinct_items) >= 10:
            return order.total() * Decimal('0.07')
        
        return Decimal(0)
{% endhighlight %}

We define a global list of promotions in **promos** which specifies via typing that it only accepts a list of promotions. The `best_promo` function takes an order and applies each promotion in the global list to it in turn and returns the max discount applicable to the order.

Run-time type checking such as `isinstance` can be enabled by defining `runtime_checkable` decorator via typing module:

{% highlight python %}
from typing import Protocol, runtime_checkable


@runtime_checkable
class Promotion(Protocol):
    def __call__(self, order: Order) -> Decimal: ...
{% endhighlight %}

We can now compare that the functions added to the promos global are indeed instances of Promotion:

{% highlight python %}
print(isinstance(promos[0], Promotion)) # returns True
{% endhighlight %}

The full example code for this article is as follows. We use dataclasses to define the Customer, Order and LineItem classes

{% highlight python %}
from typing import Protocol, Optional, runtime_checkable
from decimal import Decimal
from dataclasses import dataclass
from collections.abc import Sequence


@dataclass
class Customer:
    name: str
    fidelity: int


@dataclass
class LineItem:
    product: str
    quantity: int
    price: Decimal

    def total(self) -> Decimal:
        return self.price * self.quantity


@dataclass(frozen=True)
class Order:
    customer: Customer
    cart: Sequence[LineItem]
    promotion: Optional['Promotion'] = None

    def total(self) -> Decimal:
        totals = (item.total() for item in self.cart)
        return sum(totals, start=Decimal(0))

    def due(self) -> Decimal:
        if self.promotion is None:
            discount = Decimal(0)
        else:
            discount = self.promotion(self)
        return self.total() - discount

    def __repr__(self):
        return f'<Order total: {self.total():.2f} due: {self.due():.2f}>'
    

@runtime_checkable
class Promotion(Protocol):
    def __call__(self, order: Order) -> Decimal: ...


promos: list[Promotion] = []


def best_promo(order: Order) -> Decimal:
    return max(promo(order) for promo in promos)


class Fidelity:
    def __call__(self, order: Order) -> Decimal:
        if order.customer.fidelity >= 1000:
            return order.total() * Decimal('0.05')
        
        return Decimal(0)


class BulkItem:
    def __call__(self, order: Order) -> Decimal:
        discount = Decimal(0)
        for item in order.cart:
            if item.quantity >= 20:
                discount += item.total() * Decimal('0.1')
        
        return discount


class LargeOrder:
    def __call__(self, order: Order) -> Decimal:
        distinct_items = {item.product for item in order.cart}
        if len(distinct_items) >= 10:
            return order.total() * Decimal('0.07')
        
        return Decimal(0)


if __name__ == "__main__":
    promos.extend([Fidelity(), BulkItem(), LargeOrder()])

    print(promos)
    print(type(promos[0]))
    print(isinstance(promos[0], Promotion))

    joe = Customer('John Doe', 0)
    cart = [
        LineItem('banana', 4, Decimal('.5')),
        LineItem('apple', 10, Decimal('1.5')),
        LineItem('watermelon', 5, Decimal(5))
    ]

    order = Order(joe, cart, best_promo)
    print(order)

    # Test for fidelity promo
    joe.fidelity = 1000
    order = Order(joe, cart, best_promo)
    print(order)

    # Test for bulk item promo
    cart.append(LineItem('bananas', 21, Decimal('.5')))
    order = Order(joe, cart, best_promo)
    print(order)
{% endhighlight %}

By using Protocols, we managed to decouple the promotion functions from the codebase. If this were defined using inheritance via abstract base classes, we would need to create a custom class for each promotion, thereby creating a hierachy of inheritance. In this case, there is no clear relationship between the orders and promotions apart from during the checkout process and since promotions can change over time, using protocols allow us to decouple the implementation of the promotions away from the underlying base classes. To remove a promotion, we just remove it from the promos list.

In addition, we can utilise python's type hints and external type checkers such as mypy. For the example above, we could run mypy as so:

{% highlight python %}
pip install mypy

mypy protocols_example.py # => Success: no issues found in 1 source file
{% endhighlight %}

In this post, I aim to explain what python protocols are at a high level and provide a simple implementation of its usage. Future posts will attempt to highlight more advanced use cases of Protocol.