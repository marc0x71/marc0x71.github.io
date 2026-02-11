+++
title = " Switching from C/C++ to Rust: the invisible checklist "
date = 2026-02-02
description = "After years of writing code in C and C++, switching to Rust forced me into a reflection I wasn't expecting. It's not about syntax, performance, or language features. It's about something more subtle: where I spend my mental energy while programming."
[taxonomies]
tags = ["rust", "c", "cpp", "programing"]
[extra]
comments = true
+++

After years of writing code in C and C++, switching to Rust forced me into a reflection I wasn't expecting. It's not about syntax, performance, or language features. It's about something more subtle: **where I spend my mental energy while programming**.

## The invisible checklist

When you write in C or C++, there's a checklist you have to keep running in your head at all times. It's not written anywhere, it doesn't show up in any warning, but it's there:

- Could this pointer be null?
- Did I free this memory? Did I free it twice?
- Will this reference still be valid when I use it?
- Is this cast safe?
- Am I invoking undefined behavior without knowing it?

It's an **invisible** burden. You don't perceive it as an obstacle because it's become the norm. It's the constant background noise of every line of code you write. And the most insidious problem is that feedback, when it comes, comes **late**: a crash in production, a bug report weeks later, a sanitizer telling you that the code that "worked" didn't actually work at all.

Or worse: it never comes. The code runs, seems to work, but it's invoking undefined behavior that just happens not to manifest. Until it does.

## The "no" that comes immediately

I'll admit my early days with Rust weren't exactly smooth. The borrow checker kept stopping me. Things I would have written in thirty seconds in C++ required rethinking, restructuring, sometimes complete rewrites of my approach.

But then I realized something: that "no" coming immediately was **a gift**.

When the compiler stops you, something important happens:

1. **You're still in the mental context of the problem.** You don't have to reconstruct weeks later what you were thinking when you wrote that code.

2. **You can fix it while you're still reasoning.** The solution is still fresh, the alternatives are still visible.

3. **You learn why that thing is problematic.** The error message (often surprisingly helpful) explains *what's* wrong and *why*.

It's the difference between a teacher who corrects you while you speak and one who gives you a grade three months later without explaining where you went wrong.

## Two types of muscle memory

There's another difference I noticed, more subtle.

In C++, the muscle memory you develop over time is **avoidant**. You learn *not to do* certain things. It's a negative memory, not in a pejorative sense, but in the sense that you have to remember what *not* to do rather than what to do: a list of patterns to avoid, of traps to dodge. And this list keeps growing, because the language doesn't prevent you from falling into the traps, you just have to remember they exist.

In Rust, muscle memory is **constructive**. You learn patterns that are *inherently* correct. You don't have to remember what to avoid because the compiler won't let you do it. Instead of thinking "I must remember not to leave the door open", you learn to build a door that closes by itself.

This frees up mental space. Space I can use to think about domain logic, architecture, the problem I'm actually trying to solve, instead of tripping over my own mistakes.

## It's not about "better" or "worse"

I'm not saying C or C++ are wrong languages, or that people who use them are making a mistake. They have their legitimate use cases, mature ecosystems, decades of existing code that works.

The question I asked myself is different: **where do I prefer to spend my cognitive energy?**

Some programmers prefer total freedom and accept the responsibility that comes with it. That's a legitimate choice. Others, and I count myself among them, prefer tighter constraints that free up mental bandwidth to focus on other things.

The cost exists in both cases. In C++ it's distributed over time, invisible, perpetual. In Rust it's concentrated at the beginning, explicit, and decreases as you internalize the patterns.

For me, the second trade-off is more favorable. Not because it's objectively better, but because it fits better with how my head works, and with the kind of mistakes I tend to make.

## A final thought

What surprised me most about Rust wasn't the speed, or the language features, or the ecosystem. It was the **feeling** of programming.

That invisible checklist I'd been carrying in my head for years? It's gone. Or rather: it's been externalized. The compiler holds it for me.

And this, in the end, made me realize how heavy it was to carry.

---

*Translated to English with the help of AI.*
