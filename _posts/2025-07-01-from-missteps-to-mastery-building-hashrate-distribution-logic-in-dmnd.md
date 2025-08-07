---
layout: post
title: "From Missteps to Mastery: Building Hashrate Distribution Logic in DMND"
date: 2025-07-01
author: Ayush Bansal
categories: [Demand, Stories]
image: 2025-07-01-from-missteps-to-mastery-building-hashrate-distribution-logic-in-dmnd.jpg
---

![DMND Hashrate Distribution](https://miro.medium.com/v2/resize:fit:640/format:webp/0*rIOfI5zag_gQA1XJ.jpg)

> “The best way to learn how to build the right thing… is by building the
> wrong thing first.”

Over the last few weeks, I’ve been contributing to Demand CLI (DMND), an open-
source Stratum V2 proxy built in Rust for Bitcoin miners. What seemed like a
simple feature request — distributing mining workload across multiple upstream
pools — turned into one of the most educational and humbling experiences I’ve
had as an open-source contributor.

In this post, I’ll walk you through:

  * What I initially built (and why it was wrong),
  * The discussion with my mentor that changed everything,
  * How I fixed it with a clean architecture and dynamic assignment logic,
  * And the lessons I’m carrying forward from this experience.

# 🌐 What is Demand CLI?

Demand CLI is a lightweight Rust-based proxy that allows Bitcoin miners to
connect to the **Demand Pool** , even if their hardware still speaks the older
Stratum V1 protocol. It handles:

  * 🧠 Protocol translation (Stratum V1 ⟶ Stratum V2)
  * 🛠️ Job Declaration (letting miners propose block templates)
  * ⚡ Performance improvements with modern async networking

In short, it makes it easier for legacy miners to participate in modern,
decentralized mining with full-featured coordination.

# ❌ PR #71: When You Build the Wrong Thing (Even If It Works)

The original feature request was to allow **multi-upstream pool support** —
where miners can split their effort across different mining pool endpoints.

So I did what I thought was logical:

> _“Let’s split the hashrate. If a miner has 100 TH/s and the user configures
> a 70:30 split between Pool A and Pool B, I’ll send 70 TH/s to A and 30 TH/s
> to B.”_

I implemented this logic in PR #71. It compiled. It ran. I even wrote solid
routing logic and connection handling.

But it was wrong.

# 💣 The Problem

A miner can only open **one connection at a time**. There’s no way to “split”
a miner’s hashrate across multiple pools — at least not without firmware-level
changes that weren’t feasible.

Instead, the real requirement was something completely different.

# 💡 Mentor to the Rescue: “You’re Splitting Hashrate — But You Should Be
Splitting Miners”

My mentor helped me realize the core misunderstanding during a quick Gmeet
session:

> _❗ We don’t want to split the_ hashrate _of individual miners.  
>  ✅ We want to distribute the _miners themselves _across multiple upstream
> pools based on configured percentages._

For example:

> _If you have 10 miners and configure a 70:30 split →  
>  → 7 miners should connect to Pool A  
>  → 3 miners should connect to Pool B_

Each miner should connect entirely to **one** pool — and we aim to maintain
the configured distribution across the whole fleet.

This clarity changed everything.

# 🧱 The Rebuild: PR #100 and a Fresh Architecture

I scrapped the earlier approach and started from scratch in PR #100. This new
version correctly implemented **miner-based distribution** , using a cleaner,
smarter architecture.

Here’s the configuration format:

    
    
    pool_addresses = [  
      "stratum+tcp://pool1.example.com:3333",  
      "stratum+tcp://pool2.example.com:4444"  
    ]
    
    
    hashrate_distribution = [70.0, 30.0] # 70% to Pool1, 30% to Pool2

Now, when miners start up, they are **assigned to one of the pools based on
this distribution** , and the system keeps track of how many miners are
connected to each one.

# ⚙️ Smart Dynamic Assignment in Action

Instead of statically assigning the first `n` miners to Pool1 and the rest to
Pool2, I wrote a **dynamic routing algorithm** that adjusts in real time.

Here’s a sample output:

    
    
    // Miner 1: Pool1 needs 70%, → Pool1 selected  
    // Miner 2: Pool1 has 100%; Pool2 needs 30% → Pool2 selected  
    // Miner 3: Pool1 has 50%; Pool2 has 50% → Pool1 selected  
    // Miner 4: Pool1 has 66.66%; Pool2 has 33.33% → Pool1 selected  
    // Miner 5: Pool1 has 75%; Pool2 has 25% → Pool2 selected

✨ This ensures:

  * Pool shares are respected **even as miners come online out-of-order**.
  * No single pool gets overloaded.
  * Distribution is smooth and proportional.

The routing logic checks current miner counts for each pool, compares with the
target percentages, and routes each new miner to the pool that’s furthest
behind its goal.

# 🧠 Key Takeaways

# 1\. Understand Before You Implement

I jumped into code with a vague idea of what “multi-pool support” meant. That
cost time and effort. Now, I always **discuss the architecture with my
mentor** before writing code.

# 2\. Dynamic Is Better Than Static

By implementing a real-time assignment system, we gained flexibility,
resilience, and smarter resource utilization — which is essential for real-
world mining farms.

# 3\. Mistakes Make You Better

My first PR was wrong. It was closed. But that process taught me more than if
I had shipped the right thing on the first try. That’s open source — learning
in public, and getting better with each iteration.

# 🧭 Final Thoughts

The PR is still a **work in progress** , but the direction is now correct —
and I’m confident it’ll soon be merged.

If you’re new to open source or systems programming, let me leave you with
this:

  * Ask early.
  * Validate your understanding.
  * Don’t be afraid to scrap and rebuild.
  * Communicate often with your mentor or team.

And remember: **getting it wrong at first is often the fastest way to get it
right in the end.**

🙌 Thanks to my mentor for helping me grow through this process. If you’re
working on mining protocols, Rust projects, or just love contributing to infra
— let’s connect!
