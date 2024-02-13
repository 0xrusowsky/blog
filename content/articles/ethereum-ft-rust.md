---
title: "Rust Ethereum"
date: 2024-02-12T19:54:00+00:00
slug: ethereum-ft-rust
category: article 
summary: Introduction to the Ethereum ft. Rust series
description:
cover:
  image: covers/reth.png
  alt:
  caption: 
  relative:
showtoc: false
draft: false
---

Hi everyone!

Several months back, I decided to start learning Rust. I was mainly attracted by the fact that most of the gigachad devs of the space seemed to be in love with it. Don't get me wrong, I was also interested in its reputation for blazing-fast performance, safety, and concurrency. If Rust's features were so good, and blockchain devs were getting pilled, it must have been a sign that by learning it, I would be able to upskill and push the boundaries of my development skills, right?

So, I decided to implement the [EVM from scratch, in Rust](https://github.com/0xrusowsky/evm-from-scrustch/tree/main). While my implementation was minimal and far from perfect—gas calculations still pending—it worked! The library was capable of processing bytecode and manipulating data from the execution environment, load and store data from/into memory or storage, transfer ether, deploy new contracts, etc. After my initial implementation was completed, I decided to look into [reth](https://github.com/paradigmxyz/reth/tree/main). Despite the similarities, as expected, I could directly tell that the `reth` repo was on a completely different level. Its implementation was way more modular, defining - and leveraging- many more custom types and traits. It became clear that to improve, I wouldn't just achieve it by coding more; I also needed to start incorporating best practices into my work.

I figured that most developers who were new to Rust might face a similar challenge. Therefore, I teased on Farcaster about the possibility of creating a written series to explore how Rust intersects with Ethereum projects. The reception from the community was both surprising and motivating. It became evident that there's a collective eagerness to discover how Rust can be leveraged in the Ethereum ecosystem. Inspired by this, I've decided to embark on a series that delves into this exciting intersection.

# Series Focus: Upskill by learning from the Best

The core mission of this series is to deeply understand how seasoned Rust developers tackle the unique challenges of blockchain technology. This isn't just about diving into code; it's about understanding the technical topics behind the implementations, so that we can then comprehend the design and implementation choices that were made by those who've mastered the art of Rust in blockchain. Through this exploration, I aim to bridge theoretical knowledge with practical application, learning firsthand from the work of experienced devs.

# Join the Exploration!

This series is an invitation to join me on this educational journey, where we aim to enhance our Rust skills and apply them to the ever-evolving world of Ethereum.

Stay tuned for the first chapter of the series, kicking off our exploration of the Ethereum Virtual Machine (EVM) with Rust. This series is about collective growth, sharing discoveries, and navigating the exciting challenges of integrating Rust with blockchain technology.

---

🔔   _I will be sharing the articles on both [Farcaster](https://warpcast.com/0xrusowsky.eth) and [Tiwtter](https://twitter.com/0xrusowsky), so make sure to follow me if you are interested!_