---
layout: post
title: Write some bad code! Live a little!
---

In 2013, when I was just starting to figure out how to build iOS apps for a living, I built a two-player word game called Switchboard. I recently looked over that old code out of curiosity, and what I found was, predictably, hilarious and unrecognizable. The general style is…unconventional, to say the least; anything that needs to be used in multiple places is basically just globally accessible; and really, most of the app is contained within two view controllers, which are 2500 and 3500 lines long, and one of which is named “ViewController”. 

It’s not great!

In the past couple years, I’ve been quite focused on becoming a better software engineer. Simply put, I’ve been trying hard to go from “I can write code” to “I can write code well.” I think it’s safe to say that’s not a journey that will ever really have an end-point — the more you know, the more you realize you don’t know — but it is one that’s absolutely worthwhile. Usually, in app development, writing code to solve a problem isn’t the biggest challenge in and of itself; the challenge is in writing code that solves a problem quickly and efficiently, that can be simply understood and reasoned about by others, and that can be easily extended when the problem inevitably changes or grows in the future. 

Of course, this is all pretty obvious to any developer who’s been at it for a while. We all agree this stuff is important, and we all talk about it all the time. Design principles, architectural patterns, programming paradigms — if you want to be good at programming, you’ve got to learn this stuff, and be able to make use of the right tools in the right situations. You’ve also got to stay on top of the daily deluge of new (and recycled) tools and ideas, but not get carried away chasing after every shiny new thing that comes along.

That’s all well and good, and I agree. Nonetheless, the perspective I’m about to offer is: *forget that noise! Let’s build some cool shit*.

Switchboard’s code base was a total cluster-cuss. On the other hand: I had an idea for a game, I designed the game, I built the game, I shipped the game, and I learned a ton a long the way. On the whole, this whole process was a) a lot of fun, and b) at least partially responsible for getting me some great jobs. The code sucked, but the end result was entirely positive.

I promise I’m not suggesting that writing good code is a waste of time. In fact, as software continues to devour the world, there’s a good case to be made that we all need to get a whoooole lot better at writing good code, pronto. And, look, my weird little game was kind of a mess, was only played by a handful of people, and was never maintained — hardly an example to hold up and say “Look! Code quality is irrelevant!” But, there was also a sort of creative purity and joy to the whole thing that I’m feeling a little bit nostalgic for, and that I think has sometimes gotten left behind on my journey from “I can write code” to “I can write code well.”

This journey is fraught with countless moments of feeling like an imposter, feeling incompetent, feeling afraid to make the wrong decision or to reveal a knowledge gap in front of your peers. It can be paralyzing, and in my experience, creatively draining. And I guess what I’m getting at is this: sometimes, it’s okay to stop worrying about whether you should be applying that pattern you read about in that blog post, or about whether you’ve truly grokked what dependency inversion *actually* means, or about the fact that you still haven’t really had time to figure out what SwiftUI is all about. Instead, focus on the inherent joy of using a computer to create something out of nothing — which is likely what got you into this whole thing to begin with.

That’s really the balance I’m trying to find here, for myself and anyone else who needs to hear it. Let’s keep challenging ourselves to learn and improve as developers, but let’s not lose sight of how dang cool it is to be able to get a computer to do crazy things in the first place. Dev culture can be ruthlessly opinionated, and it can all get pretty overwhelming when you’re trying to learn something new or difficult. What I’ve seen this lead to, in myself and others, is:

1. a tendency to over-plan. Spending weeks fretting over which “architectural pattern” to use may not be the best use of time. And despite your best planning efforts, let’s be honest: you’re probably going to need to refactor a bunch of stuff in a few months anyways. 
2. a tendency to value the code itself over the end goal. Code should be easy to reason about. If you can do that, you’re probably good — even if you weren’t quite able to boil your work down into one big fancy chain of higher order functions.
3. a tendency to worry destructively about our merit or ability. Everyone feels like a bit of an imposter, and no one is as smart as they pretend to be. You’re doing great!

At their worst, these tendencies can actively slow us down or prevent us from building cool stuff, and can totally obscure the pure creative joy of seeing an idea come to life on screen. That’s a bummer. And if any of this sounds familiar, it might be a good time to put away the books and the blogs and the podcasts for a little while, and focus on writing code for the simple sake of creating something new.

Of course, writing bad code can eventually trap you in a pit of technical debt. Of course, the realities of working as part of a team or on a schedule require us to be vigilant about planning and writing good code. And of course, creativity and well-written code are not at all mutually exclusive. Of course, of course, of course. I’m not advocating for writing bad code; I’m simply advocating for anyone out there who knows what they want to build but is stuck worrying about exactly how to go about it. Go! Build a thing! There’s no single best way to do what you want to do anyway. You’re probably more capable than you think. And — quite honestly — you can always go back and change things later.

Just, please: no 3500 line view controllers.