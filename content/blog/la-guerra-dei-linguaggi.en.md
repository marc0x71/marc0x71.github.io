+++
title = "Rust, C, and C++: Beyond the Language Wars"
date = 2026-02-11
description = "There's a genre of content that populates the tech web with an almost _liturgical_ regularity: the comparison between programming languages. 'X vs Y: which to choose?', 'Why I abandoned X for Y', 'X is dead, long live Y'. Titles crafted to generate reactions, heated comments, outraged shares. And it works. It works incredibly well. But let's pause for a moment to ask ourselves: who really benefits from this narrative? Certainly not developers, who find themselves trapped in sterile debates where professional identity merges with tool choice. Not companies, which need pragmatic decisions based on real context and constraints. Not the growth of the tech community as a whole."

# draft=true
[taxonomies]
tags = ["thoughts", "cpp", "c", "rust"]
[extra]
comments = true
+++

## The Religious Wars of the Digital Age

There's a genre of content that populates the tech web with an almost _liturgical_ regularity: the comparison between programming languages. "X vs Y: which to choose?", "Why I abandoned X for Y", "X is dead, long live Y". Titles crafted to generate reactions, heated comments, outraged shares.

And it works. It works incredibly well.

But let's pause for a moment to ask ourselves: who really benefits from this narrative? Certainly not developers, who find themselves trapped in sterile debates where professional identity merges with tool choice. Not companies, which need pragmatic decisions based on real context and constraints. Not the growth of the tech community as a whole.

The real winners are the platforms. Every flame war on Reddit, every contentious thread on Twitter, every YouTube video with a provocative title generates traffic, engagement, time spent on the platform. And where there's attention, there are advertising clicks. The attention economy feeds on conflict, and language wars are low-cost conflict: passionate, endless, and fundamentally harmless to those who host it.

**These are the religious wars of the digital age**. And like historical religious wars, they produce a lot of noise, a lot of tribal belonging, and very little enlightenment.

After all, the IT world has always been fertile ground for this type of conflict. Vim versus Emacs, Java versus C#, tabs versus spaces, Linux versus Windows, Angular versus React. The list is long, and it gets longer every year. It's almost a recognizable pattern now: every time two valid alternatives emerge to solve the same problem, factions form, barricades go up, and the **holy war** begins.

Those who have worked in the field for a few years recognize the script from the first act. Yet, predictably, we fall for it again. Perhaps because defending our tools is a way of defending our choices, our invested time, our professional identity.

But this is psychology, not engineering.

## The Problem with the Wrong Question

"Which language is better?" is a poorly posed question. It's like asking which tool is better between a hammer and a screwdriver. The sensible answer is always the same: it depends on what you need to do.

Every programming language is born in a specific historical moment, to answer specific problems, with a worldview that reflects the priorities and constraints of that era. C was born in the '70s, when computational resources were extremely scarce and a minimal abstraction over hardware was needed. C++ was born in the '80s, when growing software complexity required more powerful abstraction tools. Rust was born in 2010, when decades of security vulnerabilities had made it evident that certain types of errors needed to be prevented at the language level.

None of these languages is "better" than the others. Each represents a different answer to a different question, formulated in a different context.

## Three Philosophies, Not Three Competitors

What I find most interesting, and most useful, is studying these languages as expressions of different problem-solving philosophies.

`C` embodies a radical trust in the programmer. It gives you direct access to the machine, hides nothing from you, and in return expects you to know exactly what you're doing. It's a pact between adults: maximum freedom, maximum responsibility. This philosophy has produced operating systems, databases, and infrastructures that have been supporting the digital world for decades.

`C++` starts from the same base but adds the ambition of abstraction. It wants to allow you to build complex concepts (classes, hierarchies, templates) while maintaining the performance of code close to the machine. It's a language that has accumulated tools for thirty years, becoming enormously powerful and, inevitably, enormously complex.

`Rust` takes a different path. It observes the decades of bugs, vulnerabilities, and undefined behaviors that have plagued C and C++, and decides that certain guarantees cannot be left to the programmer's discipline. They must be enforced by the compiler. It's a philosophy that sacrifices a bit of immediate freedom in exchange for safety.

## A Reflection on Kinship

Here I come to a personal consideration that made me think. Rust is often compared with C++, probably because they compete in the same application domain: high-performance systems, browsers, game engines, critical infrastructure.

But at the level of data philosophy, Rust seems closer to C to me. Both have an almost spartan approach: data is central, operations act on data, there are no complex class hierarchies or inheritance. In Rust, classes don't exist in the traditional object-oriented sense. There are structures and implementations, a mental model that a C programmer would recognize immediately.

C++, on the other hand, has embraced object orientation deeply, with all that entails: multiple inheritance, virtual functions, dynamic polymorphism. It's a powerful paradigm, but it's also a paradigm that Rust has explicitly chosen not to adopt.

This doesn't make one approach better than the other. It simply means that Rust and C share a certain worldview, data-oriented, transparent, without hidden magic, that distinguishes them from more idiomatic C++.

## Gaining Value Instead of Taking Sides

My philosophy is simple: live and let live. Every language has its reason for being, its community, its ideal use cases. Taking sides in a "faction" means precluding the possibility of learning.

And there's much to learn. Studying how Rust manages memory has made me more aware of the risks I take when writing C++. Understanding C's simplicity reminds me not to over-engineer solutions when it's not needed. Appreciating C++ abstractions shows me patterns I can bring to other contexts.

Programming languages are tools, yes, but they're also crystallizations of ideas. They're the result of intelligent people who have thought long and hard about how to solve difficult problems. Every language contains lessons, if we're willing to listen instead of judge.

## Conclusion: Beyond Stadium Cheerleading

The next time you see a title that invites you to take sides in a language war, pause for a moment. Ask yourself who really benefits from that click, from that outraged comment, from that time spent defending your favorite tool?

The real value doesn't lie in choosing the "right" language. It lies in understanding why different answers exist to the same question, and in enriching your way of thinking through this understanding.

Languages pass, evolve, sometimes die.

The ideas they contain, those remain.

---

*This article was translated from Italian using artificial intelligence.*
