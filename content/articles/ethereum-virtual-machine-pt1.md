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
4. **Increment**: Otherwise, simply advance the PC to the next opcode. _Note that some opcodes may require inputs, and that the PC can increment by more than 1 at a time._

For instance, let's disassemble `0x600f8060093d393df36000356020350160005260206000f3` into bytes. We will assume that the opcode `0x60` consumes 1 byte, and that there are no jump instructions within this bytecode.

```md
Bytecode: 60 0f 80 60 09 3d 39 3d f3 60 00 35 60 20 35 01 60 00 52 60 20 60 00 f3
      PC: 1  1  3  4  4  6  7  8  9  10 10 12 13 13 15 16 17 17 19 20 20 22 22 24
```

This structured process ensures that, irrespective of the underlying hardware or any asynchronies, VMs produce consistent outcomes when executing the same bytecode, provided that the execution environment remains unchanged.

# Meet the Ethereum VM

Ethereum, taken as a whole, can be viewed as a transaction-based state machine. In other words, a valid state transition is able to modify the state of the chain by carrying arbitrary computations. The execution model specifies how the system state is altered given a series of bytecode instructions and a small tuple of environmental data.

As described in [Ethereum's Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf), the EVM is the core of the execution layer. Providing a reproducible runtime environment that ensures unanimous agreement across the network on computation outcomes —which is key for the integrity of the blockchain—.
Its design allows for the execution of smart contracts across distinct computing environments, thanks to a sandboxed environment tailored to Ethereum's needs:
- **Ethereum-Specific Features**: Such as unique storage models, execution contexts, and essential elliptic-curve signature functions for hashing.
- **Gas Mechanism**: Solving the halting problem by metering computation, ensuring all operations are finite and predictable.
- **Cross-Contract Communication**: Facilitating interactions between contracts via call and delegatecall mechanisms.

## EVM Basics

The following section aims to explain the theory behind each of its components, as well as showcasing the core (an abstraction of cherrypicked snippets) of their `revm` implementation. The idea is to replicate the building blocks that a developer would code when implementing the EVM from scratch.

The EVM is a stack-based machine that operates with a 1024-item-deep stack, where each item is a 256-bit word. It follows a [big endian](https://developer.mozilla.org/en-US/docs/Glossary/Endianness) byte ordering convention.

{{< figure src="/blog/images/revm/architecture.png" align=center caption="_EVM architecture and brief overview of its components._" >}}

### Byte Primitive Types

Bytes can be implemented as bit vectors `Vec<u8>`. If they have a predefined number of bits (i.e. addresses) as fixed-length arrays `[u8; N]`.
The ethereum <> rust ecosystem relies on 2 main crates to handle byte primitive types: [ruint](https://github.com/recmo/uint) and [alloy-primitives](https://github.com/alloy-rs/core/). These crates define several types that reduce the burthen of having to work with bytes, with built-in arithmetic and bitwise operations, as well as seamingless conversion from one type to another. The most common types are:
```rs
// Wrapper type around bytes::Bytes to support “0x” prefixed hex strings.
// For simplicity, you can think of bytes::Bytes as an optimized impl of Vec<u8>.
pub struct Bytes(pub Bytes);

// A byte array of fixed length ([u8; N]).
pub struct FixedBytes<const N: usize>(pub [u8; N]);

// 256-bit (32-byte) unsigned integer. An EVM word.
pub type U256 = Uint<256, 4>;

// 256-bit fixed-byte integer. An EVM word.
pub type B256 = FixedBytes<32>;

// An Ethereum address, 20 bytes in length.
pub struct Address(pub FixedBytes<20>);

// An Ethereum event log object.
pub struct LogData { pub data: Bytes }

// A log consists of an address, and some log data.
pub struct Log<T = LogData> {
    pub address: Address,
    pub data: T,
}
```
As previously said, all these types come with convenient methods. [Check the docs](https://docs.rs/alloy-primitives/latest/alloy_primitives/) for further details.

### Stack

The stack is a linear data structure that operate in a last-in, first-out (LIFO) manner, where elements are added (pushed) and removed (popped) from the end of the sequence (top of the stack).

```rs
/// EVM interpreter stack limit.
pub const STACK_LIMIT: usize = 1024;

/// EVM stack with [STACK_LIMIT] capacity of words.
#[derive(Debug, PartialEq, Eq, Hash)]
pub struct Stack {
    /// The underlying data of the stack.
    data: Vec<U256>,
}

impl Stack {
    /// Instantiate a new stack with the [default stack limit][STACK_LIMIT].
    pub fn new() -> Self {
        Self {
            // SAFETY: expansion functions assume that capacity is `STACK_LIMIT`.
            data: Vec::with_capacity(STACK_LIMIT),
        }
    }

    /// Push a new value onto the stack.
    /// If it exceeds the stack limit, returns `StackOverflow` error and
    /// leaves the stack unchanged.
    pub fn push(&mut self, value: U256) -> Result<(), InstructionResult> {
        // allows the compiler to optimize out the `Vec::push` capacity check
        assume!(self.data.capacity() == STACK_LIMIT);
        if self.data.len() == STACK_LIMIT {
            return Err(InstructionResult::StackOverflow);
        }
        self.data.push(value);
        Ok(())
    }

    /// Removes the topmost element from the stack and returns it, or
    /// `StackUnderflow` if it is empty.
    pub fn pop(&mut self) -> Result<U256, InstructionResult> {
        self.data.pop().ok_or(InstructionResult::StackUnderflow)
    }
}
```

### Memory

During execution, the EVM utilizes **volatile memory**, functioning as a **word-addressed byte array** that resets after each transaction. This memory is linear and can be addressed by bytes (8 bits) or words (32 bytes or 256 bits), facilitating flexible data manipulation.

```rs
/// A sequential memory shared between calls, which uses
/// a `Vec` for internal representation.
/// A [SharedMemory] instance should always be obtained using
/// the `new` static method to ensure memory safety.
#[derive(Clone, PartialEq, Eq, Hash)]
pub struct SharedMemory {
    /// The underlying buffer.
    buffer: Vec<u8>,
    /// Memory checkpoints for each depth.
    /// Invariant: these are always in bounds of `data`.
    checkpoints: Vec<usize>,
    /// Invariant: equals `self.checkpoints.last()`
    last_checkpoint: usize,
    /// Memory limit. See [`CfgEnv`](revm_primitives::CfgEnv).
    #[cfg(feature = "memory_limit")]
    memory_limit: u64,
}

impl SharedMemory {
    /// Creates a new memory instance that can be shared between calls.
    /// The default initial capacity is 4KiB.
    pub fn new() -> Self {
        Self::with_capacity(4 * 1024) // from evmone
    }

    /// Creates a new memory instance that can be shared between calls with the given `capacity`.
    pub fn with_capacity(capacity: usize) -> Self {
        Self {
            buffer: Vec::with_capacity(capacity),
            checkpoints: Vec::with_capacity(32),
            last_checkpoint: 0,
            #[cfg(feature = "memory_limit")]
            memory_limit: u64::MAX,
        }
    }

    /// Returns a byte slice of the memory region at the given offset.
    /// Panics on out of bounds.
    #[cfg_attr(debug_assertions, track_caller)]
    pub fn slice(&self, offset: usize, size: usize) -> &[u8] {
        let end = offset + size;
        let last_checkpoint = self.last_checkpoint;

        self.buffer
            .get(last_checkpoint + offset..last_checkpoint + offset + size)
            .unwrap_or_else(|| {
                debug_unreachable!("slice OOB: {offset}..{end}; len: {}", self.len())
            })
    }

    /// Sets the given U256 `value` to the memory region at the given `offset`.
    /// Panics on out of bounds.
    #[cfg_attr(debug_assertions, track_caller)]
    pub fn set_u256(&mut self, offset: usize, value: U256) {
        self.set(offset, &value.to_be_bytes::<32>());
    }

    /// Set memory region at given `offset`.
    /// Panics on out of bounds.
    #[cfg_attr(debug_assertions, track_caller)]
    pub fn set(&mut self, offset: usize, value: &[u8]) {
        if !value.is_empty() {
            self.slice_mut(offset, value.len()).copy_from_slice(value);
        }
    }
}
```

### Storage

The EVM also has a **non-volatile storage** model where each contract keeps relevant information for the system state. The storage layout is like a hashmap: uses **key-value pairs** to access **persistent** data. At the time of writting, contracts can only interact with their own storage.

As per [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153), after the Cancun hard fork, the EVM will also implement **transient storage**. A new type of data storage mechanism that identical to the regular storage, but which is **discarded after every transaction** (only persists within a transaction). Its main application being cheaper reentrancy locks.

The EVM's deterministic nature ensures that Ethereum operates as a network with a state transition function. Given a current state and a series of transactions, it deterministically transitions to a new valid state.

Transactions either initiate message calls or deploy contracts. In both cases, the stack is loaded with opcodes and data (from transaction calldata, memory, or storage) to execute instructions and transition to a new state.

{{< figure src="/blog/images/revm/transition.png" align=center caption="_EVM execution model showcasing how the different components interact with each other._" >}}

### Gas

As for computation costs, the EVM employs a mechanism pricing mechanism called gas. In order to execute a transaction, users must pay a gas fee to compensate for the computaional resources that they spend. By doing so, we can ensure that the network is not vulnerable to spam and cannot get stuck in infinite computational loops. The gas associated with each operation is different, and must be paid regardless of the outcome of the transaction, even if it reverts.

With the introduction of [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) after the London hard fork, the fees paid by users are split between a base fee and a priority fee. This change lowered the volatility in gas prices, and gave user better predictibility.

To learn more about gas fees (structure, limits, pricing, etc.) check the following [article](https://ethereum.org/developers/docs/gas).

## Execution Environment

Environmental data necessary for the execution of the state transition. Mainly regarding the executing block, and transaction.

```rs
pub struct BlockEnv {
    /// The number of ancestor blocks of this block (block height).
    pub number: U256,
    /// Coinbase or miner or address that created and signed the block.
    /// This is the receiver address of all the gas spent in the block.
    pub coinbase: Address,
    /// The timestamp of the block in seconds since the UNIX epoch.
    pub timestamp: U256,
    /// The gas limit of the block.
    pub gas_limit: U256,
    /// The base fee per gas, added in the London upgrade with [EIP-1559].
    pub basefee: U256,
    /// The difficulty of the block.
    /// Unused after Paris (AKA the merge) upgrade, and replaced by `prevrandao`.
    pub difficulty: U256,
    /// The output of the randomness beacon provided by the beacon chain.
    /// Replaces `difficulty` after Paris (AKA the merge) upgrade with [EIP-4399].
    pub prevrandao: Option<B256>,
    /// Excess blob gas and blob gasprice.
    /// Incorporated as part of the Cancun upgrade via [EIP-4844].
    pub blob_excess_gas_and_price: Option<BlobExcessGasAndPrice>,
}
```


```



BYTECODE          MNEMONIC         STACK                 ACTION
 60 00          // PUSH1 0x00       // [0x00]
 35             // CALLDATALOAD     // [number1]          Store the first 32 bytes on the stack
 60 20          // PUSH1 0x20       // [0x20, number1]
 35             // CALLDATALOAD     // [number2, number1] Store the second 32 bytes on the stack
 01             // ADD              // [number2+number1]  Take two stack inputs and add the result
 60 00          // PUSH1 0x00       // [0x0, (n2+n1)]
 52             // MSTORE           // []                 Store (n2+n1) in the first 32 bytes of memory
 60 20          // PUSH1 0x20       // [0x20]
 60 00          // PUSH1 0x00       // [0x00, 0x20]
 f3             // RETURN           // []                 Return the first 32 bytes of memory

```