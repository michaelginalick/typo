---
title: "Knowledge in the age of AI"
date: "2026-02-20T12:00:55-06:00"
summary: "Instant answers"
description: "How to tell good from bad?"
toc: false
readTime: true
autonumber: true
math: false
tags: ["Go", "AI", "par"]
showTags: true
hideBackToTop: false
---

"We are now entering into a new phase," sings Dan Bejar. These lyrics ring more true now than maybe ever. By definition, technology is change. However, with the onslaught of AI, this change is being accelerated more than ever. So what does it mean to be a software developer in 2026? It depends on who you ask, I suppose. If you've had the displeasure of being on LinkedIn, you will find an endless feed of hype men and women encouraging anyone employed in technology to enroll in a trade school, as AI will inevitably take their job. Maybe not in 2026, but without a doubt in 2027. However, in the meantime, subscribe to their content or pay for their course, and they will help steer you through these winds of change.

"Adapt or perish" comes to mind. AI is here, and it's unlikely to go away. Even when the massive subsidies for licenses go away and companies have to begin making money, AI will still exist. And it will become more useful. This is another step in moving away from programming a machine and toward interacting with an interface. The interface can be a programming language, WYSIWYG, or now an agent. A friend of mine told me a story about how his father was forced to start using C by his then-employer. His father was incensed, asking how he was supposed to work with variables if he didn't know which register they were stored in. I don't think it is an insult to assume that many individuals employed as developers today don't know much about registers (this includes myself). The average developer 30 years ago knew much more about computing than today. They had to, as there was no other way of doing the work. Today, abstractions exist that didn't back then.

At a recent company event, we were informed that our most valuable skill set is not writing code. It is reviewing code and determining if that code is good. Setting aside that reviewing code is one of the more difficult parts of software development, it begs the question: "What is good?" This question has been on my mind a lot recently. In the age of AI, what is knowledge? Regardless of the answer to the previous question, I will continue writing code. Of course, I will use AI where it makes sense, but it is unlikely that I will tell it to write an application or a package. Most likely, I will ask it to review and refactor code I wrote. Of course, this is for personal projects. In my professional life, I may not have a choice. Maybe this is hubris? It is not that I think I can write better code than AI; it's almost certain I cannot. It's more that I want to *know* things. I don't want to *need* AI. I don't want it to *think* for me. Like a muscle you no longer exercise, if AI does everything for you, you will atrophy. Then, when you need to call on that muscle, it will no longer work the way it once did.

What I really want to get at is that, like everything else, if you want to know more about something, you have to look for it. This is getting harder since AI will *always* give you a response. I recently removed the Reddit app from my phone because it was no longer useful to me; it never was. My typical flow was to see an article, open the article in a browser tab, then promptly forget about it. However, with no new tabs being added, I've been looking through old ones. One such tab was a link to a great [post](https://nsrip.com/posts/goroutineleak.html) by [Nick Ripley](https://nsrip.com/). In this post, he points to a package written by the Go team called Par. Par uses a channel in a very elegant and safe way. I've never written a queue before, but I have written plenty of concurrent code that spawns goroutines. However, this design never occurred to me, nor would it have.

If AI wrote a queue I would be willing to bet 99 out of 100 prompts would use a mutex and add additional methods and functionality that may or may not be needed. Using a mutex is not bad, nor is adding methods that occur in other queues — which AI uses to produce the new queue. And like I mentioned before, since this pattern was new to me, I would have never given it the prompt to use this pattern. However, I feel like I learned something by reviewing this code. It expanded my notion of what is possible and I feel like it was a very productive use of my time by studying it.

I even placed this package into ChatGPT and have it explain every line to me. It, of course, offered to make improvements that would make it "production-ready" and "bulletproof," but that is not what I wanted. I wanted to think about this design and understand what it is doing and why this design is different from others. In short, I wanted to *think*, not be given the *answer*.

What I love about this implementation is that it is small enough to be a gist, and easy to enhance. For example, if you needed to add a mechanism to poll the queue to determine if items are being processed, that is another simple method. This is an outlier that almost certainly won't be the output of AI since by definition AI will output what is most statistically relevant.

Ultimately AI will make getting answers trivial; it won't make asking the right questions any easier. If you know what a queue is, it will produce one for you in an instant. If you don't know what a queue is, it is unlikely it will tell you what you need is a queue. Maybe I am wrong, but I am sticking with continuing to obtain knowledge. I will still attempt to study "good code" and read books about software development if for nothing more than for the joy of learning.

**In the interest of full transparency, I did use AI to correct any spelling and grammatical errors in this post before it was published.