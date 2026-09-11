---
title: "My Coding Workflow with an AI Pair Programmer"
date: 2026-01-29T22:57:00Z
summary: "How I use an AI coding assistant to accelerate development, from planning to implementation and verification."
tags: ["ai", "workflow", "productivity", "pair-programming", "coding"]
categories: ["Coding"]
---

Over the past few months, I've been experimenting with using an AI coding assistant as a pair programmer. What started as curiosity has evolved into a fundamentally different way of working. Here's how I structure my sessions to get the most out of this collaboration.

---

## The Three Phases

My workflow follows a consistent pattern: **Planning → Execution → Verification**. The AI assistant naturally fits into each phase, often switching between them as needed.

### Phase 1: Planning

Every significant task starts with exploration. I describe what I want to accomplish, and the AI:

1. **Explores the codebase** - It scans relevant files, understands the existing architecture, and identifies patterns already in use
2. **Researches options** - For unfamiliar technologies or approaches, it searches for best practices
3. **Proposes a plan** - It creates an implementation plan I can review before any code is written

This planning phase has saved me countless hours of "oops, I should have done it differently" moments. By reviewing the plan upfront, I catch architectural mistakes before they become expensive refactors.

```markdown
# Example Implementation Plan Structure

## Problem
What we're solving and why

## Proposed Changes
### Component 1
- File modifications with rationale

### Component 2  
- New files to create

## Verification Plan
- How we'll test it works
```

### Phase 2: Execution

Once I approve the plan, the AI writes the code. What I've learned:

- **Be specific about preferences** - If I want a particular library or coding style, I say so upfront
- **Small iterations work better** - Breaking large tasks into smaller chunks produces better results
- **It remembers context** - Within a session, it tracks what's been done and adjusts accordingly

The AI handles the tedious parts—boilerplate, configuration files, migrations—while I focus on reviewing the logic and making design decisions.

### Phase 3: Verification

After implementation, verification happens automatically:

- **Runs tests** - Unit tests, build commands, linters
- **Visual checks** - For web work, it can navigate the browser and confirm UI changes
- **Documents the work** - Creates a walkthrough of what was accomplished

---

## What Works Well

### Complex Refactors
Renaming across a codebase, moving functionality between modules, updating patterns consistently—these used to be tedious multi-hour tasks. Now they're 15-minute conversations.

### Learning New Technologies
When I need to use a framework or tool I'm unfamiliar with, I describe what I want and review the implementation. It's like having a colleague who happens to know every library.

### Documentation
I notoriously postpone writing READMEs and comments. Now I just ask, and it generates documentation that I edit for accuracy.

---

## What Requires Caution

### Security-Sensitive Code
I always manually review authentication, authorization, and data handling. The AI is helpful, but security requires human judgment.

### Business Logic
The AI doesn't know my business domain. For complex business rules, I provide detailed context or write that code myself.

### Performance-Critical Sections
When performance matters, I profile and benchmark myself rather than trusting generated code to be optimal.

---

## My Prompting Style

Over time, I've developed habits that produce better results:

1. **Start with context** - "I'm building a personal finance tracker" helps more than jumping straight to "add a button"
2. **Explain the why** - "Users need to see historical trends" leads to better solutions than just "add a chart"
3. **Be explicit about constraints** - "This must work with our existing auth system" prevents rewrites
4. **Ask for options** - "What are different approaches to X?" before committing to one

---

## The Bottom Line

Using an AI pair programmer hasn't replaced my skills—it's amplified them. I think less about syntax and more about architecture. I spend less time on boilerplate and more on the parts that require human judgment.

The key insight: treat it as a capable junior developer who types faster than anyone you've met. Give clear direction, review the output, and you'll be surprised how much ground you can cover in a single session.
