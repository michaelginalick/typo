---
title: "User defer in novel ways"
date: 2024-11-13T09:25:56-06:00
summary: "Using common language features in novel ways"
description: "Using defers in novel ways"
toc: false
readTime: true
autonumber: true
math: false
tags: ["go", "defer"]
showTags: true
hideBackToTop: false
---

Go is often described as easy to learn but difficult to master, this rings true for me as I continue to work with the language. One such example is the `defer` keyword. I've seen and used `defer` many times. Like many things, when you see it enough, it starts to feel mundane. For example, using `defer` to parse JSON into a struct is quite common.

```golang

type User struct {
  Name string
  Age  int
}

resp, err := fetchSomeData()
if err != nil {
  // handle error
}

defer resp.Body.Close()
body, _ := io.ReadAll(res.Body) // ignore error

var u User
err = json.UnMarshal(body, &u)

```

Or maybe it's used in the context of working with files.

```golang
file, err := os.Open("somefile.txt")
if err != nil {
  //handle error
}
defer file.Close()

```
In some ways, this routine approach is the essence of Go. There's nothing particularly interesting or novel about the code above; most languages have something similar. It's basically a way of saying, "execute this statement when the function completes".

However, this week, an unexpected use case for defer came up and made me see it in a new light. In a previous post, I mentioned a feature I was working on where the [chain of responsibility](https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern) design pattern was a perfect fit. In this pattern, each function is responsible for doing whatever work it needs to do, as well as calling the next item in the chain. At first, a typical function in the chain looked something like this:

```golang

func (m DoubleAgeModifier) Handle() {
	m.p.Age *= 2

	m.PersonModifier.Handle()
}

```
We do some basic operation and then call the next handler. Note that, in code not listed here, the `Handle` method is essentially calling `linkedList.Next()`. This worked well for most functions. However, some functions were more complicated. For example, they needed to traverse a structure before making the transformation or call some dependency for data. In the context of this chain, none of the individual functions return an error. If anything goes wrong, they log the error and call the next handler. Some of the functions started to look like this:

```golang

func (m ComplicatedPersonModification) Handle() {
	res, err := doSomeComplicatedThing(m.p)
  if err != nil {
    log.Errorf("An error occured %w", e)
    return m.ComplicatedPersonModification.Handle()
  }

  newRes, err := doSomeOtherComplicatedThing(res, m.p)
  if err != nil {
    log.Errorf("An error occured %w", e)
    return m.ComplicatedPersonModification.Handle()
  }

	m.ComplicatedPersonModification.Handle()
}

```

This became much more difficult to reason about for obvious reasons. This is where `defer` came in handy. Instead of calling the next handler N times, it could be placed at the top of the function once, like so:

```golang
func (m ComplicatedPersonModification) Handle() {
  defer m.ComplicatedPersonModification.Handle()

  res, err := doSomeComplicatedThing(m.p)
  if err != nil {
    log.Errorf("An error occured %w", e)
    return
  }

  newRes, err := doSomeOtherComplicatedThing(res, m.p)
  if err != nil {
    log.Errorf("An error occured %w", e)
    return
  }
}
```

Maybe it's silly, but this really floored me. While this code may not be perfect, in my opinion it's much easier to understand. I love finding new ways to use seemingly mundane language features.
