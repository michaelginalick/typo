---
title: "Wiki Links"
date: "2025-04-23T16:11:13-05:00"
summary: "You have to look at it from all angles says the cubist judge from cubist jail"
description: ""
toc: false
readTime: true
autonumber: true
math: false
tags: ["cyclic graph, Golang, wikipedia", "The Go Programming Language"]
showTags: true
hideBackToTop: false
---

The unpleasant truth about being a software developer is that you spend a lot more time reading code than writing it. In fact, sometimes the best fix is to delete code, not add more. In my experience, the bigger the company, the less code you’ll actually write.

So how do you keep your skills sharp when you’re not writing much code at work? There are a few ways. Personally, I’ve been writing these blog posts as a way to convince myself—and others—that I have an employable skill set.

One common piece of advice is to contribute to open-source projects. That’s probably the best route, but it can feel overwhelming. There are literally millions of open-source projects out there, and many have a high barrier to entry. You can easily spend hours combing through code, just trying to find the bit you want to tweak.

And to be clear, that’s not a complaint. It’s actually what a lot of developers do for a living: dive into unfamiliar codebases, try to make sense of them, and hope the fog clears eventually. The advantage of contributing to open source is that at least you get to choose the project.

The process usually looks like this: read the code, understand the parts you care about, make your changes, and—hopefully—write a lot of tests to prove your contribution(s) work and you didn't break anything in the process. That’s a huge part of the job. But then there’s the question of what makes code "good"? How do you even know if the code you’re reading is good? Honestly, I'm not sure anyone knows.

To navigate that uncertainty, I focus on books. If someone took the time to write something down, and someone else thought it was good enough to publish, chances are it’s more worthwhile than the first link you’ll find on Google or Chat GPT.

Take The Go Programming Language by Alan A. A. Donovan and Brian Kernighan. Yes, that Kernighan—co-author of the very very famous The C Programming Language. This book is full of thoughtful examples, and it’s widely considered a great intro to Go. One program that comes up multiple times is a web crawler. It does a great job of showing how natural concurrency feels in Go, and how functions can be passed around like variables. It’s a fun example—one that many would call “good code.”

I don’t know if I’d write it the same way, but maybe that’s the point. When you read other people’s code, you get to see their approach. And you carry that insight with you into your own work.

When I was going through the crawler example, I remembered a conversation I once had with a former coworker. He told me the Philosophy page on Wikipedia is the most cited page across the site — it’s basically Wikipedia’s most popular page, within Wikipedia.

For some reason, that conversation popped into my head, and it gave me an idea. I didn’t rewrite the program to prove or disprove the claim. Instead, I modified the crawler to tell me how many degrees of separation there are between two Wikipedia pages. Basically, a version of the infamous Six Degrees of Kevin Bacon.

The core functionality which I took (read: stole) from the book is as follows:

```golang
resp, err := http.Get(givenURL)

if err != nil {
        return nil, err
}

defer resp.Body.Close()
if resp.StatusCode != http.StatusOK {
    return nil, fmt.Errorf("failed getting %s: %s", givenURL, resp.Status)
}

doc, err := html.Parse(resp.Body)

if err != nil {
    return nil, fmt.Errorf("failed parsing %s as HTML: %v", givenURL, err)
}


var urls = []*url.URL{}
visitNode := func(n *html.Node) {
    if n.Type == html.ElementNode && n.Data == "a" {
        for _, a := range n.Attr {
            if a.Key != "href" {
                continue
            }

            url, err := resp.Request.URL.Parse(a.Val)
            if err != nil || url.Host != scopedHost {
                continue // ignore bad and non-wikipedia URLs
            }
            urls = append(urls, url)
        }
    }
}

forEachNode(doc, visitNode, nil)

func forEachNode(n *html.Node, pre, post func(n *html.Node)) {
    if pre != nil {
        pre(n)
    }

    for c := n.FirstChild; c != nil; c = c.NextSibling {
         forEachNode(c, pre, post)
    }

    if post != nil {
        post(n)
    }
}

```

This is a great example of the use of a first class function which Go provides. You can see `visitNode` is a fuction that gets passed around and is called on each found link that is found in the provided html. The result is a recursive fuction that requests a web page, parses the returned html, extracts all the links to other pages and returns them. This section of the book focuses on graph traversal if you're curious why the function parameters are named `pre`, and `post`.

Likewse this code is called in another bit of code that is as follows:

```golang
for i := 0; i < app.ThreadCount; i++ {
    go func() {
        for link := range app.UnseenLinks {
            foundLinks := app.Crawl(link)
            go func() { app.Worklist <- foundLinks }()
        }
    }()
}
```

This code will limit the currency to whatever is set by the app, loop through all the unseen links and add them to the queue to be processed. These examples are of course limited and incomplete. If you want to full examples, you can read the book!

The larger point is that this is code you could find in any Go code base. Taking the time to understand it is time well spent. Taking it and making something out of it is also a fruitful exercise. It was a fun project. I got to read code I didn’t write, figure out what mattered, change it in a meaningful way, and make it my own.

Here is a link to what I made. It's a fun little program that serves no real purpose other than to make me happy. For example, how many wiki links is Canadian singer [Dan Behar](https://en.wikipedia.org/wiki/Dan_Bejar) from [Philosphy](https://en.wikipedia.org/wiki/Philosophy)? Keep in mind this is not the shortest path between the two, just a path. Regardless, let's find out.

```bash
./main -sink https://en.wikipedia.org/wiki/Philosophy -source https://en.wikipedia.org/wiki/Dan_Bejar
```

```bash
FOUND THE LINK IN count=152
```

You can find the project [here](https://github.com/michaelginalick/wiki-links). Feel free to try it yourself. Keep in mind that these requests are hitting wikipedia and there is a very real chance they block your IP address if their services are abused. 

Also, please consider donating to [wikipedia](https://donate.wikimedia.org/w/index.php?title=Special:LandingPage&country=US&uselang=en&wmf_medium=spontaneous&wmf_source=fr-redir&wmf_campaign=spontaneous) to keep this amazing resource free and available for everyone.