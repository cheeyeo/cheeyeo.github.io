---
layout: post
show_meta: true
title: Generics in Go Lang
header: Generics in Go Lang
date: 2024-08-01 00:00:00
summary: What are generics and how they work in Go Lang
categories: go-lang generics 
author: Chee Yeo
---


Go-lang is a statically-typed language, which means that all variables and function parameters must be pre-defined else it will raise a compilation error.

The standard approach of developing reusable code that can accept different types would be to duplicate the same code but with different input types e.g. a function that performs division with 2 integers would need to be re-written to accept 2 float64 parameters. In fact, this is how some of the built-in library functions such as `map`, `reduce` and `filter` were initially developed by duplication to support different types of slices. Another approach could be to use `reflection` but it would incur additional computational costs if type checking has to be invoked for every function call for every type.

Generics were developed to solve this issue. Generics allows for code reuse and provides mechanism to implement data structures which could be used for multiple types. In addition, using Generics also allows you to detect incompatible types at runtime.

A good example of a data structure would be a Stack. A stack has a LIFO ( Last-In First-Out ) order where data inserted last is removed first. Assuming we have an initial stack type that accepts integers:

{% highlight golang %}
type Stack struct {
    vals []int
}

func (s *Stack) Push(val int) {
    s.vals = append(s.vals, val)
}

func (s *Stack) Pop() (int, bool) {
	if len(s.vals) == 0 {
		var zero int
		return zero, false
	}
	top := s.vals[len(s.vals)-1]
	s.vals = s.vals[:len(s.vals)-1]
	return top, true
}

func main() {
	s := Stack{}
	s.Push(10.1)
	s.Push(20)
	fmt.Println(s)
	s.Pop()
	s.Pop()
	fmt.Println(s)
}
{% endhighlight %}

The code above can be found at [this go playground](https://play.golang.com/p/9ByXBu2d_Se)

If the stack were to provide support for float types, we would need to duplicate and implement the stack to hold float types. This is error-prone and not DRY.

Generics can be used in the following contexts in go:
* custom types and data structures
* interfaces
* functions

To create a generic Stack:

{% highlight golang %}
type Stack[T any] struct {
	vals []T
}

func (s *Stack[T]) Push(val T) {
	s.vals = append(s.vals, val)
}

func (s *Stack[T]) Pop() (T, bool) {
	if len(s.vals) == 0 {
		var zero T
		return zero, false
	}
	top := s.vals[len(s.vals)-1]
	s.vals = s.vals[:len(s.vals)-1]
	return top, true
}

func main() {
	var s Stack[int]
	s.Push(10)
	s.Push(20)
	fmt.Println(s)
	fmt.Println(s.Pop())
	fmt.Println(s.Pop())
	fmt.Println(s.Pop())

	var s2 Stack[float64]
	s2.Push(10.123)
	fmt.Println(s2)
}
{% endhighlight %}

The code above can be found at [this go playgound](https://play.golang.com/p/xipIhHafm9z)

There are 2 built-in generic types we can use: `any` or `comparable`. `any` refers to any unspecified type supported by the language. `comparable` refers to any type that implements a comparison operator such as the equality operator. To use the stack to store integers, we define a stack via `Stack[int]` and a stack of floats can be defined as `Stack[float64]` without having to duplicate the stack code.

In addition, we get validation for free. Given the above example, if we accidentally tried to add a float into the first stack, we will see an error:

{% highlight golang %}
cannot use 3.333 (untyped float constant) as int value in argument to s.Push (truncated)
{% endhighlight %}

Generics can also be applied to functions. For example, suppose there is a requirement to define a generic function that accepts an int or float parameter only and doubles its value.

We define a custom interface that accepts only numeric types. The generic function is defined to only accept the custom interface as the generic type:

{% highlight golang %}
import (
	"fmt"
)

type AllowedTypes interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~float32 | ~float64
}

func Doubler[T AllowedTypes](t T) T {
	return t * 2
}

func main() {
	fmt.Println(Doubler(10))
	fmt.Println(Doubler(-3.65))
}
{% endhighlight %}

The code above can be found at this [go playground](https://play.golang.com/p/2kzb_fBBJPo)

Another example is to apply generics to interfaces. For example, we can create a generic interface called Printable that accepts a type which implements fmt.Stringer and can only be an underlying type of int, float64, or Person:

{% highlight golang %}
import (
	"fmt"
	"strconv"
)

type Printable interface {
	~int | ~float64 | Person
	fmt.Stringer
}

type PrintableInt int

func (p1 PrintableInt) String() string {
	return strconv.Itoa(int(p1))
}

type PrintableFloat float64

func (p2 PrintableFloat) String() string {
	return fmt.Sprintf("%f", p2)
}

type Person struct {
	age  int
	name string
}

func (p Person) String() string {
	return fmt.Sprintf("Person Name:%s, Age: %d", p.name, p.age)
}

func TestPrintable[T Printable](t T) {
	fmt.Println(t)
}

func main() {
	var i PrintableInt = 10
	TestPrintable(i)

	var j PrintableFloat = 3.1412
	TestPrintable(j)

	person := Person{age: 10, name: "Simon"}
	TestPrintable(person)
}
{% endhighlight %}

In the example above, we define a Printable interface which only accepts int, float64 and a custom Person struct. We also stipulated that the types must implement the fmt.Stringer interface, which means the types must implement the String() function that returns a string. We define custom integer and float64 types that implement the String() function. The code can be accessed via this [go playground](https://play.golang.com/p/gC0dR7SjOQo)


In this short post, I aim to introduce the use of generics in go-lang to create reusable data structures and functions. Generics is an advanced subject and readers are encouraged to explore more by reading the go lang blog to find out more.