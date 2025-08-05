---
title: "Go channel usage"
date: "2025-07-29"
summary: "Working from a mutex to a channel"
description: "Understanding where and how to use channels correctly"
toc: false
readTime: true
autonumber: true
math: false
tags: ["concurrency", "Go", "channels"]
showTags: true
hideBackToTop: false
---

I was reading this [post](https://bitfieldconsulting.com/posts/iterators) the other day -  mostly because I forgot the syntax for iterators and needed an overall refresher on the topic. There was a function I was writing that was initially written to return a slice of values. It then dawned on me that it didn’t need to wait until all the values were ready. By using iterators, the method could be ranged over in a cleaner way and they could begin being consumed as soon as the first value is available. However, a particular section of the post stood out to me:

> Looping over this channel with range behaves exactly the way you’d expect, 
which is to say just the same as with an iterator. But there are at least two things about this solution that aren’t ideal. One is that your program is now concurrent,
which means it’s probably incorrect. (Prove me wrong, Gophers. Prove me wrong.)

> Concurrent programming is just hard, and there are many easy-to-make 
mistakes that can cause a concurrent program to deadlock, crash, and so on. 
It’s not a tool we deploy lightly, and it wouldn’t be worth the extra
complexity just to iterate over a slice.

If consulting is your line of work, it probably behooves you to write statements like these. By all accounts, the folks at Bitfield are very knowledgeable, and I enjoy their explanations and examples. But is it presumptuous to say that all concurrent programs are wrong? Probably. I guess it depends on what you mean by “wrong.”

Still, it got me thinking about my own understanding of channels and concurrent programming. Having used Go (and other languages) for a while now, I’d like to think I more or less know what I’m doing. I’m certainly not the best programmer you’ll meet, but I like to think I fall somewhere in the fat part of the bell curve.

With all that said, I don’t use channels very often. In fact, I could probably count on both hands the number of times I’ve reached for them. Yes, they were all in the context of concurrent programs, and yes—maybe they were "wrong." Actually, I’m not sure about that last part. I hope not, but there's always a possibility.

There are lots of examples on how to use channels - they are just values after all. I am thinking of one such [example](https://go.dev/blog/pipelines) that I've read many times. This post is often sited as a must for anyone wanting to get started with channels. I've never written a pipeline so this doesn't quite hit home. 

What made channels click for me was to take an existing piece of concurrent code and modify it to use channels instead of a mutex. Below is a somewhat simplified version:

```golang

package main

import (
	"fmt"
	"log"
	"math/rand"
	"time"
)

// Parallelize run jobs in parallel
func Parallelize(jobs ...func() error) []error {
	var wg sync.WaitGroup
	var mu sync.Mutex
	errs := []error{}
	for _, job := range jobs {
		wg.Add(1)
		go func(j func() error) {
			defer wg.Done()
            mu.Lock()
			defer mu.Unlock()

			err := j()
			if err != nil {
				errs = append(errs, err)
			}

		}(job)
	}
	wg.Wait()

	return errs
}

func main() {
	jobs := []func() error{}

	for i := 0; i < 100; i++ {
		jobs = append(jobs, func() error {
			delay := rand.Intn(100)
			log.Printf("job %d delay %d", i, delay)
			time.Sleep(time.Duration(delay) * time.Millisecond)

			if i % 5 == 0 {
				return fmt.Errorf("%d failed", i)
			}

			return nil
		})
	}

	if errs := Parallelize(jobs...); len(errs) > 0 {
		log.Printf("\n %d jobs failed\n", len(errs))
		for _, err := range errs {
			log.Printf("%v", err)
		}
	}
}
```

Depending on your definition of “wrong,” you might consider this program incorrect. At the very least, it can be improved. For example, the `Parallelize` function can be made lock-free by using a channel instead of a mutex. Channels handle synchronization internally:

```golang
func Parallelize(jobs ...func() error) []error {
	var wg sync.WaitGroup
	errs := make(chan error, len(jobs))
	resErr := []error{}
	for _, job := range jobs {
		wg.Add(1)
		go func(j func() error) {
			defer wg.Done()
			if err := j(); err != nil {
				errs <- err
			}

		}(job)
	}

	go func() {
		wg.Wait()
		close(errs)
	}()

	for err := range errs {
		resErr = append(resErr, err)
	}

	return resErr
}
```

This change is pretty straightforward. The most subtle part is the goroutine that waits for all jobs to finish and then closes the channel. Closing channels correctly can be tricky. Since they're heap-allocated, a premature exit can leave goroutines or resources hanging, leading to memory leaks.

Before moving forward, we can change the function to return the channel itself instead of collecting the errors in a slice. The benefit is that the caller can start processing errors immediately as they arrive:

```golang
func Parallelize(jobs ...func() error) <-chan error {
	var wg sync.WaitGroup
	errs := make(chan error, len(jobs))
	for _, job := range jobs {
		wg.Add(1)
		go func(j func() error) {
			defer wg.Done()
			if err := j(); err != nil {
			    errs <- err
			}

		}(job)
	}

	go func() {
		wg.Wait()
		close(errs)
	}()

	return errs
}
```

You could also keep the error slice and return an iterator, but I'm trying to write about channels so a channel can be returned.

In a perfect world, this implementation is fine. Once all functions are processed (fanned out), the channel closes and everything winds down nicely. Unfortunately, real-world programs need to handle restarts and unexpected shutdowns.

To avoid goroutine leaks, we need a way to signal cancellation. Imagine a scenario where your function is processing work, but the server receives a shutdown signal.

Currently, there’s no way for your function to know that. We need a mechanism to “stop short.” The typical tool for this is the [context package](https://pkg.go.dev/context). In general, any function that spawns goroutines should accept a context.Context as its first parameter. Using the `Parallelize` nction from earlier, the updated signature becomes: `Parallelize(ctx context.Context, jobs ...func() error)`


This allows the server to signal cancellation, and for jobs to respond accordingly. Here's the updated implementation using `select`:

```golang
func Parallelize(ctx context.Context, jobs ...func() error) <-chan error {
	var wg sync.WaitGroup
	errs := make(chan error, len(jobs))
	for _, job := range jobs {
		wg.Add(1)
		go func(j func() error) {
			defer wg.Done()

			select {
			case <-ctx.Done():
				return
			default:
				if err := j(); err != nil {
					errs <- err
				}
			}

		}(job)
	}

	go func() {
		wg.Wait()
		close(errs)
	}()

	return errs
}
```

With all these updates, we now have something that’s approaching a reasonably correct program. However, there are still important considerations. Although `goroutines` are very lightweight (each starts with a 2KB stack), it’s not always a good idea to spawn them uncontrollably. For example, what happens if this function is called with millions of jobs? As currently implemented, there's a one-to-one mapping between jobs and `goroutines`. It may be wise to throttle the number of concurrent `goroutines` to avoid exhausting system resources.

Another concern is what happens if the server receives a shutdown signal—jobs that were in progress or queued could be lost. Ideally, you'd implement some kind of write-ahead log or job persistence mechanism to ensure reliability in the face of failure. These topics, however, are beyond the scope of this post.

Back to the original point of this post—introducing concurrency requires a lot of care. The authors at Bitfield are right that these techniques shouldn’t be deployed lightly. They introduce complexity, edge cases, and are ripe for subtle bugs. If you're lucky, the program panics. But like many concurrency issues, the bugs are often silent, slipping into production and producing incorrect results that are difficult to reproduce.

Here’s the full program:

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"sync"
	"time"
)

// Parallelize run jobs in parallel
func Parallelize(ctx context.Context, jobs ...func() error) <-chan error {
	var wg sync.WaitGroup
	errs := make(chan error, len(jobs))
	for _, job := range jobs {
		wg.Add(1)
		go func(j func() error) {
			defer wg.Done()

			select {
			case <-ctx.Done():
				return
			default:
				if err := j(); err != nil {
					errs <- err
				}
			}

		}(job)
	}

	go func() {
		wg.Wait()
		close(errs)
	}()

	return errs
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	jobs := []func() error{}

	for i := 0; i < 100; i++ {
		jobs = append(jobs, func() error {
			delay := rand.Intn(100)
			log.Printf("job %d delay %d", i, delay)
			time.Sleep(time.Duration(delay) * time.Millisecond)

			if i%5 == 0 {
				return fmt.Errorf("%d failed", i)
			}

			if i%3 == 0 {
				fmt.Printf("Cancelling jobs at %d \n", i)
				cancel()
			}

			return nil
		})
	}

	for err := range Parallelize(ctx, jobs...) {
		log.Printf("%v", err)
	}
}
```