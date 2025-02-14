---
layout: post
show_meta: true
title: Testing golang 1.24.0 map updates
header: Testing golang 1.24.0 map updates
date: 2025-02-09 00:00:00
summary: Testing the new golang 1.24.0 swiss-table maps
categories: go-lang maps
author: Chee Yeo
---

[go1.24.0 Changelog]: https://go.dev/doc/go1.24

As part of a personal project, I created a simple in-memory cache which uses a map that uses a ( string, struct ) data structure to cache a complex struct in memory to be accessed via a different part of the web application on redirect.

I also read about the new swiss-table maps in [go1.24.0 Changelog] so I decided to rewrite the existing cache implementation in order to benchmark it.

The cache struct is as follows:

{% highlight golang linenos %}
package cache

import (
	"sync"
)

type Cache[K comparable, V any] struct {
	items map[K]V
	mu    sync.Mutex // mutex for controlling concurrent access to cache
}

func New[K comparable, V any]() *Cache[K, V] {
	return &Cache[K, V]{
		items: make(map[K]V),
	}
}

func (c *Cache[K, V]) Set(key K, value V) {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.items[key] = value
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()

	value, found := c.items[key]
	return value, found
}

// deletes the key value pair from cache
func (c *Cache[K, V]) Remove(key K) {
	c.mu.Lock()
	defer c.mu.Unlock()

	delete(c.items, key)
}

func (c *Cache[K, V]) Pop(key K) (V, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()

	value, found := c.items[key]
	if found {
		delete(c.items, key)
	}

	return value, found
}
{% endhighlight %}

It uses generics to create a generic cache which uses strings as key names and the any type to support any built-in or custom type. For example, to create an instance of the cache which has strings as keys and integers as values, we can create it as so:

{% highlight golang linenos %}
myCache := cache.New[string, int]()

myCache.Set("item1", 123)
{% endhighlight %}

In my use case, the cache needs to store custom structs which contain an API client. I created the following benchmark test to create a baseline on how the efficiency of the cache operations. A benchmark test is the same as an ordinary go-lang test. It's normally invoked after all the unit tests have run for a given test case. In the benchmark test, I created a single cache and separate benchmarks to test its basic operations such as get, set, pop and remove:

{% highlight golang linenos %}
package cache

import (
	"fmt"
	"os"
	"testing"

	"example.com/geminitest/internal/websockets"
)

var (
	MyCache = New[string, *websockets.Client]()
)

func TestMain(m *testing.M) {
	fmt.Println("TestMain setup...")
	for i := 0; i < 1_000_000; i++ {
		key := fmt.Sprintf("key-%d", i)
		MyCache.Set(key, &websockets.Client{})
	}
	exitVal := m.Run()
	os.Exit(exitVal)
}

func BenchmarkCacheGet(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for i := 0; i < 1_000_000; i++ {
			key := fmt.Sprintf("key-%d", i)
			MyCache.Get(key)
		}
	}
}

func BenchmarkCachePop(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for i := 0; i < 1_000_000; i++ {
			key := fmt.Sprintf("key-%d", i)
			MyCache.Pop(key)
		}
	}
}

func BenchmarkCacheRemove(b *testing.B) {
	for i := 0; i < b.N; i++ {
		for i := 0; i < 1_000_000; i++ {
			key := fmt.Sprintf("key-%d", i)
			MyCache.Remove(key)
		}
	}
}
{% endhighlight %}

Each benchmark test begins with the keyword `Benchmark`. We pass in `testing.B` parameter type which has the same functionality of a `testing.T` type but with additional support for benchmarking. Every benchmark test has a loop that iterates from 0 to `b.N` to ensure that the functions are run repeatedly with larger values of N until the timing results are accurate.

To invoke the benchmark tests within the module only, we run:
{% highlight golang linenos %}
go test -v -bench=. ./internal/cache
{% endhighlight %}

The following is a terminal output of the test results when run with go1.23.4:
{% highlight shell linenos %}
TestMain setup...
goos: linux
goarch: amd64
pkg: example.com/geminitest/internal/cache
cpu: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
BenchmarkCacheGet
BenchmarkCacheGet-12       	       4	 253498901 ns/op
BenchmarkCachePop
BenchmarkCachePop-12       	       9	 120908361 ns/op
BenchmarkCacheRemove
BenchmarkCacheRemove-12    	       9	 122953659 ns/op
PASS
ok  	example.com/geminitest/internal/cache	4.953s

{% endhighlight %}

Next, I downloaded and installed golang 1.24.0 as a separate go installation and ran the same benchmark as above:

{% highlight shell linenos %}
export GOPATH=/home/user/go
export GOBIN=/home/user/go/bin

go install golang.org/dl/go1.24.0@latest

/home/user/go/bin/go1.24.0 download

/home/user/go/bin/go1.24.0 test -v -bench=. ./internal/cache
{% endhighlight %}

The second set of test results are shown below:

{% highlight shell linenos %}
TestMain setup...
goos: linux
goarch: amd64
pkg: example.com/geminitest/internal/cache
cpu: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
BenchmarkCacheGet
BenchmarkCacheGet-12       	       4	 261347032 ns/op
BenchmarkCachePop
BenchmarkCachePop-12       	       9	 114864197 ns/op
BenchmarkCacheRemove
BenchmarkCacheRemove-12    	       9	 117305923 ns/op
PASS
ok  	example.com/geminitest/internal/cache	5.668s
{% endhighlight %}

Compared to the 1.23.0 version, the 1.24.0 version has shown slightly faster run times with the pop and remove operations being faster in terms of nanoseconds per op. However, the get operation is slower resulting in the longer overall test time. This might be due to a misconfiguration or that I'm storing structs or complex types as the map value. Future posts will aim to address this issue.

In summary, we can utilize the faster swiss-table map implementation in the latest version of go for more efficient data lookup. No firther changes are required in existing code but more benchmarks need to be run to ensure that the upgrade adds value to more efficient runtimes.