---
title: "Ethereum Virtual Machine (in Rust) - Part 1"
date: 2024-02-12T19:54:00+00:00
slug: ethereum-virtual-machine-pt1
category: article 
summary: Introduction to the EVM in Rust series
description: First chapter of the Ethereum ft. Rust series, were I explore the implementation of the EVM.
cover:
  image:
  alt:
  caption: 
  relative:
showtoc: true
draft: true
---

# What are Virtual Machines?
At their core, VMs are software emulations of physical computers. They encapsulate an entire computing environment within a layer of abstraction that runs atop physical hardware. This design allows VMs to offer a sandboxed execution environment for applications, ensuring that software runs independently of the underlying hardware specifics. As you can tell, these properties are perfect for distributed systems that aim to decentralize its execution around the globe.

*But how do they work?* VMs are sophisticated programs that execute bytecode, a form of precompiled, low-level instructions designed for efficient execution by the VM. Each instruction consists of an operation code (opcode) and its arguments, guiding how the VM manipulates data and manages operations.

The execution process within a VM follows a simple yet powerful loop, with a program counter (PC) tracking its progress through the bytecode. Here’s a basic outline of this execution loop:

1. **Fetch**: Retrieve the instruction at the position indicated by the PC.
2. **Execute**: Carry out the operation specified by the instruction, which may involve arithmetic calculations, memory access, or other data manipulations.
3. **Jump**: If the instruction involves a jump, update the PC to point to the target instruction within the bytecode.
4. **Increment**: Otherwise, simply advance the PC to the next opcode.

This structured process ensures that, irrespective of the underlying hardware or any asynchronies, VMs produce consistent outcomes when executing the same bytecode, provided that the execution environment remains unchanged.

## Meet the Ethereum VM
The EVM is the heart of Ethereum's execution layer, providing a reproducible runtime environment that ensures that all nodes in the network agree on the outcome of computations -which is crucial for maintaining the integrity and consistency of the blockchain-.

The EVM stands at the core of Ethereum's execution layer, providing a reproducible runtime environment that ensures unanimous agreement across the network on computation outcomes —which is key for blockchain's integrity—.
The EVM’s design not only allows for the execution of smart contracts across varied computing environments but also introduces a specialized, sandboxed environment tailored to Ethereum's needs:
- **Gas Mechanism**: Solving the halting problem by metering computation, ensuring all operations are finite and predictable.
- **Ethereum-Specific Features**: Such as unique storage models, execution contexts, and essential elliptic-curve signature functions for hashing.
- **Cross-Contract Communication**: Facilitating interactions between contracts via call and delegatecall mechanisms.

---

The EVM is a stack-based machine that follows a [big endian](https://developer.mozilla.org/en-US/docs/Glossary/Endianness) byte ordering convention. It operates with a **1024-item-deep stack**, where each item is a **256-bit word**.

During execution, the EVM utilizes **volatile memory**, functioning as a **word-addressed byte array** that resets after each transaction. This memory is linear and can be addressed by bytes (8 bits) or words (32 bytes or 256 bits), facilitating flexible data manipulation.

Contracts on Ethereum act like hashmaps, using **key-value pairs** for accessing **persistent** data in an account's storage layout.

{{< figure src="/blog/images/revm/evm-architecture.png" align=center caption="_EVM architecture and brief overview of its components._" >}}

The EVM's deterministic nature ensures that Ethereum operates as a network with a state transition function. Given a current state and a series of transactions, it deterministically transitions to a new valid state.

Transactions either initiate message calls or deploy contracts. In both cases, the stack is loaded with opcodes and data (from transaction calldata, memory, or storage) to execute instructions and transition to a new state.

{{< figure src="/blog/images/revm/evm-transition.jpeg" align=center caption="_EVM execution model showcasing how the different components interact with each other._" >}}

As for computation costs, the EVM employs a mechanism called gas, priced in gwei. Gas costs vary by instruction, serving as a guard against spam and ensuring efficient network operation.

# Rust feat Ethereum

## The Client Diversity Problem

As it stands, [Go-Ethereum (Geth)](https://github.com/ethereum/go-ethereum) dominates the execution layer, [commanding a 72% share](https://clientdiversity.org/#distribution). While Geth's pivotal role in Ethereum's infrastructure is undeniable, its predominance carries inherent risks. If there were to be zero-day client bug, the network would be severely impacted.

{{< figure src="/blog/images/revm/client-diversity.png" align=center caption="_Client distribution landscape of the Ethereum network._" width=200% >}}

This situation highlights the importance of diversifying the execution client ecosystem. Encouraging the adoption of minority clients by major validators (such as Lido or Coinbase) is gaining traction in the community as a means to mitigate these risks. These alternative clients not only contribute to the ecosystem’s resilience by reducing dependency on a single point of failure, but also do so by encouraging technological innovation. For instance, the innovations brought by `Reth` and `Erigon`, with its [stage-sync](https://erigon.substack.com/p/erigon-stage-sync-and-control-flows) feature, or `Reth`'s modularity, demonstrate the potential for new clients to enhance Ethereum's capabilities.

## Why Rust?

[Rust](https://www.rust-lang.org/) and [Reth (Rust-Ethereum)](https://github.com/paradigmxyz/reth/tree/main) emerge as pivotal developments in broadening Ethereum's client diversity. It is no secret that [Paradigm](https://www.paradigm.xyz/oss) has been doubling-down on rust during the last couple of years, sponsoring opens-source projects such as: [ethers-rs](https://github.com/gakonst/ethers-rs), [foundy](https://github.com/foundry-rs/foundry), [revm](https://github.com/bluealloy/revm), [alloy](https://github.com/alloy-rs/alloy), and [reth](https://github.com/paradigmxyz/reth/tree/main). Their bet is clear, to build blazing-fast ethereum software leveraging rust's memory safety, and concurrent processing.

As an aspiring crypto developer, I've pinpointed several key areas to master. Writing smart contracts with Solidity or Vyper is just the tip of the iceberg. True proficiency requires a deeper dive. A low-level understanding of the EVM is crucial, and [Huff](https://github.com/huff-language/huff-rs) is the perfect language for that. Finally, despite the most impactful actions in crypto often occurring on-chain, a well-rounded developer also needs solid foundations in computer science and distributed systems. Whether your interest lies in MEV or developing blockchain-related software, a high-performance language like `Rust` (or `Go`) is indispensable.

While `reth` is on my radar for future exploration, I've chosen to start with a focused look at one of its core crates. As hinted earlier, we will deep-dive into `revm`: a Rust implementation of the EVM. In the articles to follow, we will examine `revm`'s compliance with the Ethereum Yellow Paper and the design decisions that experienced Rust developers made.

Join me on this exploration to not only understand the intricacies of EVM implementation but also to discover how Rust’s type system can be leveraged to build modular, efficient, and flexible software.
