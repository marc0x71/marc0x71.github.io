+++
title = "Vibe coding: the program I didn't write"
date = 2026-07-14
description = "I let AI rewrite in five minutes a project I'd been thinking about for years. It worked perfectly. But something felt off."
[taxonomies]
tags = ["ia", "coding"]
+++

## The boss I always wanted to be

There's an old cliché about the kind of boss everyone recognizes: the one who walks in, says "get this done," and disappears until it's ready. They don't care about the *how*, only the result. And if the result is good, well, somehow they always end up taking a bit of the credit too.

For years I looked at this figure with a mix of annoyance and — let's be honest — a little envy. It must be nice, right? Saying what you want and watching it appear.

Then vibe coding showed up, and suddenly I could be that boss.

## The little project that had been waiting for years

Tucked away in a digital drawer — one of those folders you open once a year, usually with a mix of fondness and embarrassment — I had a small project written a while back. Python 2. Not Python 3, actual Python 2, the kind with `print` and no parentheses, just to give you a sense of how long ago that was.

It worked. It worked great, actually — functionality was never the issue. It was born the way these things are born: a prototype thrown together on the fly, because at that moment I just needed a quick solution, not something meant to last. The classic script that solves one specific problem and then just sits there, untouched, because "it works, why risk it?".

But the thought never fully went away: giving it a more solid shape, more *structured*, less "scripty." Turning it into something someone could call a *program* without ironic air quotes. Maybe rewriting it in a more suitable language — Rust, C++ — something with the kind of rigor that Python 2, by its nature and by how I'd written it back then, simply didn't have.

I thought about it now and then. Then something new would come along, something more urgent, more shiny, and the little project would go back in the drawer. As always.

## Five minutes (not timed, but I'd swear to it)

A few days ago I decided it was time. I took the code, pasted it into Claude Code, and explained what I wanted: a rewrite in C++.

Three or four clarifying questions — the right ones, for what it's worth, not random ones — and then it went quiet and got to work. I didn't time it precisely, but as I experienced it, no more than five minutes passed.

The code was ready. It compiled. It worked. There were even tests.

A job that, honestly, if I'd tackled it myself, would have taken me three or four days — between dusting off syntax, rethinking memory management with the right mindset, and the usual pile of compilation errors that accumulate in the early hours of a port — was already there, done, working.

And here's the part I didn't expect: I wasn't happy. I was sad.

## Speed isn't the problem

Let's be honest: this isn't the first time I've used AI in my work, and it's not been a bad experience overall. I use it often, and in ways I consider healthy: to stress-test an idea before diving in headfirst, to have alternative solutions suggested that I then weigh with a critical eye, to offload the tasks I'd call "necessary but tedious" — documentation, unit tests, the part of the job we know how to do but don't exactly look forward to on a Sunday evening.

In those cases, it's always felt like talking things through with an experienced colleague. One who maybe won't blow your mind with groundbreaking ideas — that's not really their strength — but one you can rely on for the fundamentals, for good practices, for not badly reinventing the wheel. A colleague you argue with, ask "what if we tried this instead?", who answers back, corrects you, confirms your thinking. A conversation.

Vibe coding is something else entirely. It's not a conversation, it's total delegation. You say what you want, and someone — something — does it for you. Not three questions followed by working together: three questions, and then the task disappears from your hands.

## The code that doesn't feel like mine

I spent a long time thinking about why that C++ rewrite, technically flawless, left me with such a strange feeling. And I think I found an answer, even if a partial one.

That code carries none of my three imaginary days of effort. There's no afternoon spent staring at an incomprehensible stack trace. There's no small but real satisfaction of finally understanding why that pointer was pointing where it shouldn't. There's not even the quiet cursing at a dumb compilation error, the kind you fix with a single comma.

There's no struggle in it, in other words. And I realized — with a certain discomfort, because it's not a comfortable conclusion — that part of the sense of ownership I feel toward the code I write comes exactly from that. From the struggle. From the time invested. From the mistakes made and fixed with my own hands, literally.

Code born in five minutes, however correct, however elegant, remains a foreign object. It works, but it doesn't feel mine.

To be fair, it wasn't all perfect: there are a few architectural choices in the rewrite that don't fully convince me — some abstractions I find excessive for the size of the project, a pattern or two applied more out of habit than real necessity. Those are exactly the points I want to revisit in my next vibe coding session, trying to actually "discuss" things with the AI instead of just accepting the output.

And here a clarification is worth making, just to call things by their name: what I did is genuine vibe coding, even by the sharp distinction that Antirez, the creator of Redis, himself draws — he reserves that term for total delegation, and calls *automatic programming* the kind of work where the programmer keeps steering every decision. I didn't steer anything. I asked, I waited, I received.

And maybe that's exactly the point: I don't miss suffering over a misbehaving pointer, or spending an afternoon staring at a stack trace. What I miss is not having built, step by step, the mental model of the program. It's during that process that a piece of software stops being merely something that works and becomes something I feel is mine.

I used to think I wanted to be that boss who says "get this done" and comes back when it's ready. Then I actually became him. And I discovered that the part I truly loved was never receiving the program. It was writing it.

---

*This translation was made with the help of AI.*
