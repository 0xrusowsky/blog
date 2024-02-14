---
title: "Ethereum, but made in Rust"
date: 2024-02-12T19:54:00+00:00
slug: ethereum-made-in-rust
category: article 
summary: Introduction to a new series that explores blockchain-related projects built with Rust.
description:
cover:
  image: covers/eth-in-rust.png
  alt:
  caption: 
  relative:
showtoc: false
draft: false
---

Hey there!

As an aspiring crypto dev, I've pinpointed several key areas to master. Writing smart contracts with [Solidity](https://soliditylang.org/) or [Vyper](https://github.com/vyperlang/vyper) is just the tip of the iceberg. True proficiency requires a deeper dive. A low-level understanding of the EVM is crucial, and [Huff](https://github.com/huff-language/huff-rs) is the perfect language for that. But you will still need more! Despite the most impactful actions in crypto often occurring on-chain, a well-rounded developer also needs solid foundations in computer science and distributed systems. Whether you are facinated by MEV, or interested in developing blockchain-related software, a high-performance language like [Rust](https://www.rust-lang.org/) or [Go](https://go.dev/) is indispensable.

Several months back, I decided to start learning rust. I was mainly attracted by the fact that most of the gigachad devs that I admire seemed to be in love with it. Everyone was highlighting its blazing-fast performance and safety, so I decided to give it a go. If rust's features were so good, and blockchain devs were getting pilled, it must have been a sign that by learning it, I would be able to upskill and level-up as a dev, right?

After reading [the rust book](https://github.com/rust-lang/book), I decided to implement the [EVM from scratch, in Rust](https://github.com/0xrusowsky/evm-from-scrustch/). While my implementation was minimal and far from perfect —gas calculations still pending— it worked! The library was capable of processing bytecode and manipulating data from the execution environment, load and store data from/into memory or storage, transfer ether, deploy new contracts, etc. After my initial implementation was completed, I decided to look into [reth](https://github.com/paradigmxyz/reth/tree/main). As expected, despite the similarities, I could immediately tell that the `reth` repo was on a completely different level. Its implementation was way more modular, defining and leveraging many custom types and traits. It became clear that if I wanted to improve, I couldn't simply keep coding more projects; I also needed to start incorporating best practices into my work.

I figured that most developers who were new to rust might face a similar challenge. Therefore, I teased on Farcaster about the possibility of creating a written series to explore how rust intersects with Ethereum projects. The reception from the community was both surprising and motivating. It became evident that there's a collective eagerness to breach the gap between "not-so-idiomatic docs" and rust learnoors that want to discover how rust can be leveraged in the Ethereum ecosystem. Inspired by this, I've decided to embark on a series that focuses on this exciting intersection.

The idea is simple: to **upskill by learning from the best**. The core mission of this series is to deeply understand how seasoned rust developers tackle the unique challenges of blockchain technology. This isn't just about diving into code; it's about understanding the technical topics behind the implementations, so that we can then comprehend the design and implementation choices that were made by those who've mastered the art of rust in blockchain. Through this exploration, I aim to bridge theoretical knowledge with practical application, learning firsthand from the work of experienced devs.

---

## Join the Exploration!

This series is an invitation to join me on this educational journey, where we aim to enhance our rust skills and apply them to the ever-evolving world of Ethereum.

Stay tuned for the first chapter of the series, kicking off our exploration of the Ethereum Virtual Machine (EVM) written in rust. This series is about collective growth, sharing discoveries, and navigating the exciting challenges of integrating rust with blockchain technology.

---

🔔   _I will be sharing the articles on both [Farcaster](https://warpcast.com/0xrusowsky.eth) and [Twitter](https://twitter.com/0xrusowsky), so make sure to follow me if you are interested!_