---
layout: post
show_meta: true
title: Using iterators in go-lang
header: Using iterators in golang
date: 2025-04-06 00:00:00
summary: How to use iterators in go-lang
categories: go-lang iterators gemini
author: Chee Yeo
---

[iter package]: https://pkg.go.dev/iter

In go 1.23, the [iter package] was added to support the use of iterators. Iterators are functions that traverse a collection such as slices and passes each successive element to a callback function called `yield` which returns the element or stops the iterator if it returns `false` such as reaching the end of the sequence.

The following examples creates an iterator that returns the elements of a slice:

{% highlight go linenos %}
func createIter(nums []int) func(func(int) bool) {
	return func(yield func(int) bool) {
		for n := range nums {
			if !yield(n) {
				return
			}
		}
	}
}
{% endhighlight %}

The return type of the iterator is of type `func(int) bool` and within the return statement, we call `yield` on each element of the input parameter, which is passed to an inner function that returns true/false on whether the element is to be returned. If the return value is false, the element will not be returned by the iterator. This is a useful pattern when we are using iterators together with slices to filter and generate a new collection, which we will show later in the post.

Iterators can only be used in traversal. We can retrieve the elements from an iterator by calling `range` on it:

{% highlight go linenos %}
func main() {
    nums := []int{1, 2, 3, 4, 5}
    iter := createIter(nums)
    fmt.Printf("%+v\n, reflect.TypeOf(iter))
    for n := range iter {
        fmt.Println(n)
    }
}
{% endhighlight %}

{% highlight go linenos %}
func(func(int) bool)

0
1
2
3
4
{% endhighlight %}

The following example uses a conditional function on the inputs within the iterator to filter out certain elements from the input. The example only wants to retrieve prime numbers from a collection. Firstly, we define a function to determine if an integer is a prime:

{% highlight go linenos %}
func isPrime(n int) bool {
	if n <= 1 {
		return false
	}
	for i := 2; i < n; i++ {
		if n%i == 0 {
			return false
		}
	}

	return true
}
{% endhighlight %}

Next, we apply this function to each element in the iterator within the `yield` statement. Recall that the any action within `yield` which returns false will cause that element to be excluded. In this example, we only allow elements that return true from calling `isPrime`:

{% highlight go linenos %}
func getPrimeNumbers(nums []int) func(func(int) bool) {
	return func(yield func(int) bool) {
		for n := range nums {
			if isPrime(n) {
				if !yield(n) {
					return
				}
			}
		}
	}
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
	for x := range getPrimeNumbers(nums) {
		fmt.Printf("x: %d\n", x)
	}
}
{% endhighlight %}

Running the example above produces only numbers which are prime:

{% highlight go linenos %}
x: 2
x: 3
{% endhighlight %}

Other packages in the standard library support the use of iterators such as `maps` and `slices`. As mentioned earlier, we can apply filtering to an input slice and return a new slice via the use of iterators. Given the same example as before, we can wrap both iteration and filtering using `yield` and `slices.Collect` to return a new slice of prime numbers:

{% highlight go linenos %}
func isPrime(n int) bool {
	if n <= 1 {
		return false
	}
	for i := 2; i < n; i++ {
		if n%i == 0 {
			return false
		}
	}

	return true
}


func main() {
    nums := []int{1, 2, 3, 4, 5}

    primes := slices.Collect(func(yield func(int) bool) {
		for _, n := range numbers {
			if isPrime(n) {
				if !yield(n) {
					return
				}
			}
		}
	})

    fmt.Println(primes) # => [2 3 5]
    fmt.Println(nums) # => [1, 2, 3, 4, 5]
}
{% endhighlight %}

`slices.Collect` takes in an input of `iter.Seq[E]` which is defined by the inline iterator function. Within the `yield` callback, we only return prime numbers that return true after calling the `isPrime` function. Note that the original slices of integers remain unchanged. This is an effective way of performing further processing without any hidden side-effects or having to modify the input in-place.

In summary, the `iter` package provides essential tools to create and use iterators in go-lang which is an essential feature of traversing data structures.
