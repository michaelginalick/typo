---
title: "Don't Kill My Vibe"
date: 2026-10-20T15:30:48-05:00
summary: "Using AI deliberately is still learning — it just looks different now."
description: ""
toc: false
readTime: true
autonumber: true
math: false
tags: ["Go", "design patterns", "vibe coding"]
showTags: true
hideBackToTop: false
---

There is a lot of revisionist history going on in the software field right now. The industry is changing no doubt. Scary things are happening. I'd be lying if I said I feel good about my future job prospects. However, the idea that the software development was in a good place before AI is simply not true.

There was an old joke (and by old I mean like 10 years ago) that anytime you came across a complicated bit of code in peer review, you would ask the person "which stackoverflow post did you get this from?". Now the collective is pearl clutching about how understanding every line of every PR was time well spent. How spending hours and hours trying to recreate a bug was a badge of honor. The idea of understanding everything begins and ends when you onboard the framework of your choice or integrate with a cloud provider. You can take this as far as you want. It's an endless slide into layers and layers of abstraction.

Many people perfected the art of blowing fog around during standup meetings as to why making a simple change was taking an entire sprint. Now AI is making the fog dissipate. You're really no longer able to do those things or won't in the very near future. For example, if you work anywhere in the vicinity of a computer it's assumed you can work Excel, Word, Powerpoint and all the other office suite of tooling. You can't shrug your shoulders and say "I'm not an Excel guy. Ask someone else.". I think the same will happen with software. You'll soon be expected to produce software regardless of your background or training.

In a recent interview, Jon Bodner, said he thinks that in the near future there will be more software but fewer software developers. This is the most realistic observation I've heard. My partner, who even conservative estimates have as about 10000x smarter than me is making software. She works with internal reporting and since it's not technically profitable and will only ever be consumed by members inside her company she vibe codes it up. She knows exactly what they need, and trying to translate the requirements to a developer would be more time consuming than just telling the agent what to produce. Does this work 100% correctly all the time? No. Of course not. Are there times when she needs to ping "real developers" to ask questions? Yes, of course. But these would exist regardless of who was doing the work. Now, if you're like me, you may be asking, "but who is going to maintain this!?". The answer to this question is no one. No human is ever going to look at this code or modify it in any way. Any new feature or bug fix will only ever be done by AI.

So where does that leave us? The people who make money writing software? To be completely honest, I have no idea. But I feel better knowing that no one really knows. People predict a huge bust and others predict a reordering of how society works. They are probably both wrong, but I rest somewhat easier knowing that no one really knows. I'm still of the mind that continuing to learn is a solid investment. Your value will still be in what you know and understand, concepts you've read or learned about and the when/where/how to deploy them. In many ways continuing to learn has never been easier. I've taken this to heart recently and have made something out of it. Well, I vibe coded many parts of it so a liberal use of the word "made". Below is a recent example of how I gathered some recent learnings into one coherent application.

I like collecting recipes. There are a few sites I like going to and finding fun things to make or at least getting ideas from. However, as by design almost all recipe websites are rendered unusable because of ads and other garbage crammed into every pixel of the page. I don't blame them, hosting sites cost money and putting ads on your page is a no-brainer to make money or at the very least break even. But I couldn't take it anymore. They're unusable. Especially on a phone where I use them most because I'm in the kitchen and don't want to haul my laptop around. This led me to do some research. All or most popular publicly accessible recipe sites use the same content format, recipe cards (visual layouts) combined with JSON-LD schema markup. This is, by design, very easy to scrape.

Lucky for me I already had a scraper whose design I took from [The Go Programming Language](https://www.thriftbooks.com/w/the-go-programming-language_brian-w-kernighan_alan-a-a-donovan/13867697/item/23690184/), and I had been reading about API design and other fun things you can do with Go so I decided to just create my own site. I used AI to create it, but I was very specific in how I wanted the application to be designed. There were patterns I wanted to implement and concepts I wanted to solidify for my own understanding and continuing growth as a developer. Here are a list of sources I used if you're interested:

- The overall API design was inspired by this post:
  - https://golangbot.com/secure-go-code/
- Separating the SQL from the application code was taken from the excellent book:
  - https://pragprog.com/titles/mtgo/effective-go-recipes/
- The web scraping algorithm was taken from another excellent book:
  - https://www.thriftbooks.com/w/the-go-programming-language_brian-w-kernighan_alan-a-a-donovan/13867697/item/23690184/
- The middleware design was taken from this post:
  - https://www.alexedwards.net/blog/making-and-using-middleware
- The Hexagonal Architecture approach came from this Udemy course:
  - https://www.udemy.com/course/rest-based-microservices-api-development-in-go-lang/

I did use AI to write this or in some cases transpose it from other sources. Rarely did I use what it first produced. I really had to force it to not reach over data boundaries. It kept telling me, "for an app this small this is over engineering". That's fine and true, but even big applications start small. Experimenting with small application design decisions will only put you in a better place to make large application design choices. The refinement process is where learning happens for me. I love fiddling with code until it seems "right". This is how I learn and will hopefully continue to remain employable. I guarantee if prompted AI to write this application, it would not come up with this approach or implement some of the more flexible patterns. Is that a good or bad thing? I can't answer that. For me, it's a good thing. Despite not writing this application in the traditional sense, I have a very real understanding of the overall patterns it uses. It's not a complicated application by any means, but I understand every line. Could I have written every line? Yes. Would it have taken me much, much, much longer, so long that it probably would have never been completed? Also Yes.

After work the other week, I was walking to the train and I heard the conversation of the people in front of me. The gist of it was basically "if the only thing you can do is use AI, then why do I need you?". I think of this often and thought about it a lot as I was making this application. Like any tool, use it, but understand what it's doing and what problem it is helping you solve. You can still learn and continue to use your skills even with AI. In fact, it's never been easier.

If you're interested in viewing the code it can be found here: https://github.com/michaelginalick/simply-scraped-recipes