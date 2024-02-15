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


## Let's switch gears. The Client Diversity Problem.

As it stands, [Go-Ethereum (Geth)](https://github.com/ethereum/go-ethereum) dominates the execution layer, [commanding a 72% share](https://clientdiversity.org/#distribution). While Geth's pivotal role in Ethereum's infrastructure is undeniable, its predominance carries inherent risks. If there were to be zero-day client bug, the network would be severely impacted.

{{< figure src="/blog/images/revm/client-diversity.png" align=center caption="_Client distribution landscape of the Ethereum network._" width=110% >}}

This situation highlights the importance of diversifying the execution client ecosystem. Encouraging the adoption of minority clients by major validators (such as Lido or Coinbase) is gaining traction in the community as a means to mitigate these risks. These alternative clients not only contribute to the ecosystem’s resilience by reducing dependency on a single point of failure, but also do so by encouraging technological innovation. For instance, the innovations brought by `Reth` and `Erigon`, with its [stage-sync](https://erigon.substack.com/p/erigon-stage-sync-and-control-flows) feature, or `Reth`'s modularity, demonstrate the potential for new clients to enhance Ethereum's capabilities.

[Rust-Ethereum (Reth)](https://github.com/paradigmxyz/reth/tree/main) emerges as pivotal development in broadening Ethereum's client diversity. It is no secret that [Paradigm](https://www.paradigm.xyz/oss) has been doubling-down on rust during the last couple of years, sponsoring opens-source projects such as: [ethers-rs](https://github.com/gakonst/ethers-rs), [foundy](https://github.com/foundry-rs/foundry), [revm](https://github.com/bluealloy/revm), [alloy](https://github.com/alloy-rs/alloy), and [reth](https://github.com/paradigmxyz/reth/tree/main). Their bet is clear, to build blazing-fast ethereum software leveraging rust's memory safety, and concurrent processing.

As a foundry maxi and long-time enjoyoor, I am certain that the rust client will succeed. I believe that the client diversity picture will be way more different in a year. Because of that, I want to improve as a developer and eventually be able to contribute to this ecosystem of projects. While `reth` is on my radar for future exploration, I've chosen to start with a focused look at one of its core crates. As hinted earlier, we will deep-dive into `revm: a Rust implementation of the EVM`. In the articles to follow, we will examine `revm`'s compliance with the Ethereum Yellow Paper and the design decisions that experienced rust developers made.

## Join the Exploration!

This series is an invitation to join me on this educational journey, where we aim to enhance our rust skills and apply them to the ever-evolving world of Ethereum. Let's discover how rust’s type system can be leveraged to build modular, efficient, and flexible software.

Stay tuned for the first chapter of the series, kicking off our exploration of the Ethereum Virtual Machine (EVM) written in rust. This series is about collective growth, sharing discoveries, and navigating the exciting challenges of integrating rust with blockchain technology.

---

🔔   _I will be sharing the articles on both [Farcaster](https://warpcast.com/0xrusowsky.eth) and [Twitter](https://twitter.com/0xrusowsky), so make sure to follow me if you are interested!_