---
layout: post
show_meta: true
title: Cancelling a pair of go routines
header: Cancelling a pair of go routines
date: 2023-05-10 00:00:00
summary: How to cancel a set of go routines if one fails
categories: go-lang
author: Chee Yeo
---

[errGroup]: https://pkg.go.dev/golang.org/x/sync/errgroup

[AWS S3 Pipes]: https://github.com/cheeyeo/AWS_S3_PIPES

In a recent go-lang project, I created a set of go routines to run conurrently. It involves setting up a pair of reader and writer processes to run a named pipe.

My initial attempt involved using `waitGroups` to run the goroutines and channels with custom wait loops to cancel the go routines if one of them fails. e.g if the data fetching fails

However this approach was error prone.

I switched to using [errGroup] which provides synchronization, error propagation and context cancelation for groups of goroutines working as subtasks on a larger task.

This means we can use the context returned to cancel the goroutine and the `Wait()` synchronization function to wait for the subtasks to complete without using wait groups.

My implementation became:
{% highlight go %}
g, ctx := errgroup.WithContext(context.Background())

pipeInput := &pipes.DownloadInput{}
pipeOutput := &pipes.DownloadOutput{File: file}

g.Go(func() error {
	c := pipes.NewPipeWithCancellation(pipeOutput)
	return c.Stream(ctx, pipe, bucket, key)
})

g.Go(func() error {
	c := pipes.NewPipeWithCancellation(pipeInput)
	return c.Stream(ctx, pipe, bucket, key)
})

if err := g.Wait(); err != nil {
	fmt.Errorf("Error in download: %v\n", err)
}
{% endhighlight %}

`errGroup.WithContext()` returns an `*errgroup.Group` struct and a context which we can use to cancel the goroutines.

We invoke the functions to run in goroutines using `g.Go`. The first call that returns a non-nil error cancels the group's context, which is exactly the pattern we want here.

The `Stream` function is implemented as so:

{% highlight go %}
func (c *PipeWithCancellation) Stream(ctx context.Context, pipe string, bucket string, key string) error {
	errChan := make(chan error)

	go func() {
		err := c.pipe.Stream(ctx, pipe, bucket, key)
		errChan <- err
	}()
	defer close(errChan)

	for {
		select {
		case err2 := <-errChan:
			return err2
		case <-ctx.Done():
			return ctx.Err()
		}
	}
}
{% endhighlight %}

We run the function in a separate goroutine which returns an error if it exists and puts it onto the `errChan` channel.

The second for loop is blocked on waiting for any errors messages from the error channel or a signal on `context.Done()`.

Given two concurrent goroutines, if the first goroutine returns an error, it cancels the group's context. This sends a signal to the context `Done` channel. The second goroutine will pick up the signal in its select block case of `<-ctx.Done()`, returning the error and stopping execution.

IMO, this is a more sustainable way of managing concurrent goroutines which are subtasks of a bigger task.

The [AWS S3 Pipes] project is available for perusal.