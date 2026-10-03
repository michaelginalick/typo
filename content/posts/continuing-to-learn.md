---
title: 'Continuing to Learn'
date: 2026-10-02T15:20:23-05:00
summary: "Is spending time learning to become a better developer still worth it?"
description: "LLMs and learning should not be mutually exclusive"
toc: false
readTime: true
autonumber: true
math: false
tags: ["Go", "AI"]
showTags: true
hideBackToTop: false
---

*I want to acknowledge that the below is geared towards business logic and the use of AI. Business logic has the luxury of being relatively simple and also unimportant. I mean, sure, if your program panics maybe someone gets their delivery one day later than it would have otherwise.*

In Noah Lawson's excellent post ["The dimished art of coding"](https://nolanlawson.com/2026/03/22/the-diminished-art-of-coding/) he states that what was once a creative form of expression is now dimished. Many people consider Donald Knuth's "The Art of Computer Programming" tantamount to a holy text. Even the name suggests creative expression. In the interest of transparency, I've never even pretended to read the TAOCP series for many reasons, the most of which is that it far exceeded my intellectual ceiling.

The reason I bring these items up is to acknowledge the fact that there has always been this tension in software development. Some consider it a craft, always striving to find better, more beautiful, efficient ways to do something. However, if you are employed as a software developer your job is to produce functioning code, not make art or develop your craft.

This idea was put rather bluntly by a co-worker of mine. He's a lead on a project I work on and what he said can be summarized as: Your job has alwys been to produce code. Whatever bells and whistles come with that are ancillary and largely unimportant. At first, I was dismissive of this. I mean, what he does is write code, what I do is much more than that. Or is it? The answer to that is a resounding No. 

Of course, with the advent of LLMs it's really not even up for debate anymore. If you want to stay employed, you need to produce code. With LLMs it's not even a matter of not knowing how to do something. Before, one could at least have the time buffer of needing time to research. That is not the case any longer. If you can describe the functionality, the LLM can do the rest.

So where does that leave things?

No one has been more guilty of playing the "software is a craft" card than me. Is software no longer a craft? In the professional world, yes, that concept no longer exists. No one cares if your code is clean or spaghetti. In fact, now the only thing that matters is that the AI agent can access your code. It will tell you what is wrong and won't bother you with patterns and abstractions. The best part is that when the spaghetti code needs to be modified in some way, the agent will do it, and more than likely do it correctly. Many would consider this a good thing! You no longer need to worry about understanding or fear touching some brittle bit of code.

But you can and should continue learning. LLMs and learning are not mutually exclusive. In fact, one could argue that we now have access to possibly the greatest learning tool in all of human existence. By way of example, in a previous post I wrote about the ["Chain of Responsibility" pattern](https://michaelginalick.pages.dev/posts/chain-of-responsibility/). The pattern was lifted almost verbatium from an excellent Udemy course I took, called [Design Patterns In Go](https://www.udemy.com/course/design-patterns-go/). 

It is almost shameful how much this course helped me and how much I took (read: stole) from it. At the time, I was writing what turned out to be a mini ETL pipeline in Go and it was not going well. Taking what I learned from this course almost single handedly turned that project around. It is hard to quantify how much this course benefited me.

However, even with the patterns I took from the course there still existed warts in the design. After leaving that job, I couldn't let it go. I wanted to make the project better. One obvious flaw with the design is how every new rule needed a bunch of boiler plate code. I think there was something like 4 individual things needed per transformation rule. Of course, this is fine with a few rules, but as more and more rules were added it became cumbersome to continue adding all the set up code.

But the core design was very good. So good, that I extracted it into an open source package. Upon doing so, I asked Claude for a review. Of course, it told me all the short comings and how it violated the DRY principle. Many of the flaws it found were things I expected. It gave me some good suggestions and other not so good suggestions. It optimized the design beyond what is reasonable - basically sacraficed readability for speed that is not needed. But in the end, it did come up with an overall better, more flexible design and API.

Obviously, this example is a simple example. If you ask AI to write the Chain of Responsibility pattern in Go, I promise it will look very different from what is in this library. I tried it! Whether that is a good or bad thing is up to the individual. In this instance I used my knowledge, which I gathered from another human being and filtered it through my existing experience to make something useful; to solve a problem.

If I didn't know about this pattern or seek to expand my skillset via Udemy, I woudn't have learned anything. I'd have the same skills I had previously. The project would have continued working but it would have been less good in almost every area.

All this is to say, as software moves more into managing agentic code, what you know is still the limiting factor. That part has not changed. A developer who spends time learning will still produce better output, even if that output is done by an agent. Software can still be as much of an art or craft as it has ever been. As Lawson states in his post, the craft aspect is still present; it's just different now.

You can find the [package](https://github.com/michaelginalick/go-chain) here if you're curious.