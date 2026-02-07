+++
title = "AI in Programming: Tool or Crutch?"
date = 2026-01-18
description = "AI coding assistants like GitHub Copilot, ChatGPT, and Claude have revolutionized software development, but are we using them wisely? This article explores the critical question: are these powerful tools helping us become better programmers, or are they becoming crutches that prevent us from developing essential skills?"

[taxonomies]
tags = ["ia", "coding"]
+++

In recent years, we've witnessed an explosion of artificial intelligence tools for software development: GitHub Copilot, ChatGPT, Claude, and many others. But as these tools become increasingly pervasive, an urgent question arises: are we using them the right way? Or are we raising a generation of programmers who know how to use AI but don't know how to program?

In this article, I want to share my perspective on where AI can be a valuable ally and where it risks becoming a crutch that hinders professional growth.

## The Problem: Outsourcing Thinking

Let's be frank: I increasingly observe programmers, especially beginners, approaching every problem by copying the request into an AI and pasting the resulting code. Zero reflection, zero understanding, zero learning. It's as if we're witnessing a sort of "outsourcing of thinking" – the problem is no longer to be solved, but to be delegated.

### An Old Problem, but Amplified

This problem wasn't born with AI. Who hasn't copied code from StackOverflow without fully understanding it? Who hasn't taken a working solution and pasted it into their project without truly understanding why their own version wasn't working?

The problem of "copy-paste without comprehension" has existed since programming forums began. The crucial difference is that back then there were **"natural brakes"** built into the process.

When you searched online, you had to:
- Read discussion threads
- Understand the context of someone else's problem
- See the conversation between the person asking and those answering
- Manually adapt the code to your specific situation

Even if you ended up copying, there was still some cognitive processing in between. You had to read, interpret, adapt.

With AI, this process has compressed to: I write the question, I receive the code perfectly formatted for my specific case, I paste it. Done. The cycle is so fast and smooth that it completely bypasses any form of reflection.

Moreover, on StackOverflow you'd find not just the code, but also:
- Explanations of why that solution worked
- Discussions in the comments about pros and cons
- Alternatives proposed by other users

It was an environment that, even unintentionally, encouraged learning through the exchange of ideas. AI gives you the direct answer, clean, without exposing you to the debate and complexity of reasoning. It's tremendously more efficient, but also tremendously more dangerous for those who are learning.

### The Calculator Analogy

Imagine an elementary school child who needs to learn addition. If every time they need to do 7 + 5 they reach for the calculator, what happens?

They'll never learn to calculate mentally. They won't develop number sense. They won't understand the underlying mathematical reasoning. And when they find themselves in a situation where the calculator isn't available, they'll be completely lost.

The same applies to programming. If every time you need to implement an algorithm, fix a bug, or structure an application you ask the AI, **you're not developing the fundamental skills of a programmer**.

### Copying Without Guarantees

There's another interesting parallel: copying homework at school. When you copy from someone, you have no guarantee that solution is correct. You might be copying from someone who knows as much as you do, or even less.

With AI it's worse: AI can generate code that compiles, that seems to work, but that hides subtle bugs, security vulnerabilities, or monstrous inefficiencies. If you don't have the skills to understand that code, how do you know if it's good or terrible?

### What We're Really Losing

When we delegate reasoning to AI, we lose three fundamental things:

**Problem-solving ability**: Programming isn't just writing correct syntax, it's knowing how to analyze a problem, break it down, find creative solutions. If AI thinks for you, you never develop this skill.

**Originality and creativity**: AI code is, by definition, a recombination of existing patterns. It can't be truly innovative. If you want to invent new solutions to old problems, you need to cultivate your own way of thinking, your unique perspective.

**Debugging and maintenance ability**: This is crucial. If you didn't write the code, if you don't understand it deeply, how do you debug it when it inevitably breaks? How do you optimize it? How do you adapt it when requirements change?

## The Correct Uses of AI

Now, careful: I'm not saying AI is useless or harmful in absolute terms. On the contrary! There are excellent ways to use these tools.

### 1. AI as a Personal Tutor

AI can be an extraordinary tutor for learning. Are you learning a new language? Ask AI to explain concepts to you, show you examples, clarify the differences between different approaches.

**Practical examples:**
- "Explain the difference between a list and a tuple in Python"
- "How does the trait system work in Rust?"
- "What are the advantages of the Observer pattern compared to the Publisher-Subscriber pattern?"

Here AI isn't writing code for you, it's acting as a teacher. The difference is subtle but crucial: you're using AI to **learn how to do** something, not to **do it instead of you**.

### 2. Understanding Classic Algorithms

Want to understand how Quicksort works? Ask AI to explain it step by step, maybe with a visual example. This is active learning, not passive delegation.

### 3. Documentation Support

Did you write a complex piece of code yourself and want to document it well? AI can help you structure the documentation, suggest what to include, even generate basic comments that you then refine.

Here the intellectual work was done by you, AI is just helping with the "boring" part of the process.

### 4. Test Generation

Did you implement a function and want to test it thoroughly? Ask AI to generate test cases, including edge cases you might not have considered.

Again: you wrote the code, you understand what it does, and you're using AI for a more complete verification. The production code remains the fruit of your reasoning.

### The Golden Rule

**Use AI to learn, understand, verify, document. Don't use it to replace your critical thinking and your ability to solve problems.**

AI should be like an advanced textbook, not like a classmate whose homework you're copying.

## Legal and Ethical Issues

There's an aspect that many underestimate, but which is fundamental: legal and licensing issues.

### The Licensing Problem

Generative AIs have been trained on enormous amounts of code available on the internet. Much of this code is open source, with specific licenses like GPL, MIT, Apache, and so on.

The million-dollar question is: **when an AI generates code inspired by code it was trained on, should that new code be considered "derived" from the original code?**

If the answer is yes, there are problems. Copyleft licenses like GPL require that derived code maintain the same license. This means that if you use code generated by an AI trained on GPL code, you might be obligated to release your entire project under GPL license.

### A Regulatory Gray Area

At the moment, this is a gray area. There are ongoing lawsuits, and the regulatory framework is evolving. Several companies have already banned the use of generative AI for production code precisely because of these risks.

If you work for a company and generate code with AI, you're potentially introducing enormous legal risks. The company could find itself exposed to litigation for copyright or license violations.

And I'm not even talking about the cases (rare but documented) where AI reproduces almost literally pieces of code from its training set, perhaps including original comments and variable names. In those cases there's no doubt: it's a direct copy.

### The Ethics of Professional Training

There's also a broader ethical issue: as a community, we have a responsibility to train competent programmers. If we normalize the use of AI as a replacement for reasoning, we're damaging the overall quality of the profession.

## Toward a Balanced Vision

AI isn't going away. In fact, it will become increasingly pervasive. And it's not even desirable to completely eliminate it from development tools. The point is **finding the right balance**.

### Four Guiding Principles

Here's my proposal for guidelines for responsible AI use:

**First principle – Learn the basics first without AI**: If you're starting out, if you're learning a new language or paradigm, do it without AI assistance. Get your hands dirty, make mistakes, debug, suffer a little. That's how you really learn.

**Second principle – Use AI to accelerate, not to replace**: Once you know how to do something, AI can help you do it faster. But understanding must come first.

**Third principle – Always verify**: Never, and I repeat never, copy code from AI without fully understanding it. Read it, analyze it, test it, verify that it does exactly what it needs to do.

**Fourth principle – AI as a competence amplifier**: The better you are, the more value you can extract from AI. An experienced programmer uses AI surgically for specific tasks. A beginner who delegates everything to AI remains a beginner forever.

### My Personal Workflow

Let me tell you how I use AI in my daily work:

**What I do:**
- I use it to explore new APIs or libraries: "How do you use this function?" or "Show me an example configuration"
- I use it for architectural brainstorming: "What are the pros and cons of this approach?" (but the final decision is always mine)
- I use it to generate basic tests, which I then expand and refine manually
- I use it for documentation, but only after the code is written and working
- If I need a classic algorithm, I ask for pseudocode or a description of how it works, not the complete implementation. This way the "transformation" into real code in my specific context remains under my control

**What I DON'T do:**
- I don't use AI to write the core logic of the application
- I don't use AI to solve complex bugs
- I don't use AI to implement algorithms I should know

### Competence is Irreplaceable

Remember: **AI is a powerful tool, but it's just a tool**. Competence, deep understanding, the ability to think critically and creatively – these things can only come from experience, study, trial and error.

There's no shortcut to becoming a good programmer.

## Conclusion

AI in programming can be a valuable ally if used to learn, deepen, verify, and document. It becomes a problem when it replaces your critical thinking and prevents you from developing real skills.

My appeal, especially to those starting out: **don't be in a hurry**. Programming is a skill that's built with time, with deliberate practice, with mistakes. Every problem you solve on your own is a brick you add to your skills.

AI will always be there, ready to help you. But first you need to know where you're going.

---
_Note: This article was translated from Italian to English using AI assistance._
