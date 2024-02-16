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

The following section aims to explain the theory behind each of its components, as well as showcasing the essence of their `revm` implementation. Because of that, some of the provided snippets may differ from the original code, aiming to illustrate the foundational building blocks a developer would create when coding the EVM from scratch. As we delve into the intricacies of the yellow paper, we will refine these blocks to match the real `revm` implementation.

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
/// A word-addressable memory, which uses a `Vec` for internal representation.
/// A [Memory] instance should always be obtained using the `new` static method
/// to ensure memory safety.
pub struct Memory {
    /// The underlying buffer.
    buffer: Vec<u8>,
    /// Memory limit. See [`CfgEnv`](revm_primitives::CfgEnv).
    #[cfg(feature = "memory_limit")]
    memory_limit: u64,
}

impl Memory {
    /// Creates a new memory instance. The default initial capacity is 4KiB.
    pub fn new() -> Self {
        Self::with_capacity(4 * 1024) // from evmone
    }

    /// Creates a new memory instance with the given `capacity`.
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
    pub fn slice(&self, offset: usize, size: usize) -> &[u8] {
        let end = offset + size;

        self.buffer
            .get(offset..offset + size)
            .unwrap()
    }

    /// Set memory region at given `offset`.
    /// Panics on out of bounds.
    pub fn set(&mut self, offset: usize, value: &[u8]) {
        if !value.is_empty() {
            self.slice_mut(offset, value.len()).copy_from_slice(value);
        }
    }

    /// Sets the given U256 `value` to the memory region at the given `offset`.
    /// Panics on out of bounds.
    pub fn set_u256(&mut self, offset: usize, value: U256) {
        self.set(offset, &value.to_be_bytes::<32>());
    }
}
```

### Storage

The EVM also has a **non-volatile storage** model where each account (contract) keeps relevant information for the system state. The storage layout is like a hashmap that uses **key-value pairs** to access **persistent** data. Each contract has its own storage and -at the time of writting- can only interact with their own storage.

```rs
/// An account's Storage is a mapping of 256-bit integer key-value pairs.
pub type Storage = HashMap<U256, U256>;
```
_*Note that `Hashmap` is a type defined in the standard collection. As such, among other convenient methods, it already has getter and setter functions. If you are not familiar with hashmaps yet, check the [standard library docs](https://doc.rust-lang.org/std/collections/struct.HashMap.html)._

As per [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153), after the Cancun hard fork, the EVM will also implement **transient storage**. A new type of data storage mechanism that identical to the regular storage, but which is **discarded after every transaction** (only persists within a transaction). Its main application being cheaper reentrancy locks.

```rs
/// Structure used for EIP-1153 transient storage.
pub type TransientStorage = HashMap<(Address, U256), U256>;
```

### World State

The world state, or state trie, is a mapping between addresses and account states (account basic info and its storage). Although it is not stored on the blockchain -only the state root is stored in the block header-, the EVM has access to all this information stored in a state database. At the time of writing, Ethereum uses a data structure called modified merkle-patricia trie, which requires full-nodes to maintain a the full chain state (not the history) on their local database. The [following image](https://i.stack.imgur.com/afWDt.jpg) is a nice visual representation of Ethereum's tries where you can easily see the relationship between the account storage trie, an account's basic info, and the world state trie.

```rs
/// EVM State is a mapping from addresses to accounts.
pub type State = HashMap<Address, Account>;

pub struct Account {
    /// Balance, nonce, and code.
    pub info: AccountInfo,
    /// Storage cache
    pub storage: Storage,
    /// Account status flags.
    pub status: AccountStatus,
}

// The `bitflags!` macro generates structs that manage a set of flags.
bitflags! {
    pub struct AccountStatus: u8 {
        /// When account is loaded but not touched or interacted with.
        /// This is the default state.
        const Loaded = 0b00000000;
        /// When account is newly created we will not access database
        /// to fetch storage values
        const Created = 0b00000001;
        /// If account is marked for self destruction.
        const SelfDestructed = 0b00000010;
        /// Only when account is marked as touched we will save it to database.
        const Touched = 0b00000100;
    }
}

/// AccountInfo represents basic account information.
pub struct AccountInfo {
    /// Account balance.
    pub balance: U256,
    /// Account nonce.
    pub nonce: u64,
    /// code hash,
    pub code_hash: B256,
    /// code: if None, `code_by_hash` will be used to fetch the code when needed.
    pub code: Option<Bytecode>,
}

impl Default for AccountInfo {
    fn default() -> Self {
        Self {
            balance: U256::ZERO,
            code_hash: KECCAK_EMPTY,
            code: Some(Bytecode::new()),
            nonce: 0,
        }
    }
}
```

To efficiently manage and store persistent data, a dedicated database is used. As `revm` aims to be a modular framework, a trait (rather than a type) is implemented. This design decision allows any user-desired db architecture to be used, as long as the `Database` trait is implemented.

```rs
/// EVM database interface.
pub trait Database {
    type Error;

    /// Get basic account information.
    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error>;

    /// Get account code by its hash.
    fn code_by_hash(&mut self, code_hash: B256) -> Result<Bytecode, Self::Error>;

    /// Get storage value of address at index.
    fn storage(&mut self, address: Address, index: U256) -> Result<U256, Self::Error>;

    /// Get block hash by block number.
    fn block_hash(&mut self, number: U256) -> Result<B256, Self::Error>;
}

/// EVM database commit interface.
pub trait DatabaseCommit {
    /// Commit changes to the database.
    fn commit(&mut self, changes: HashMap<Address, Account>);
}
```

When defining traits, it is recommended to provide different levels of mutability and ownership. `Database` requires mutable access `(&mut self)` to perform state-altering operations. On the other hand, `DatabaseRef` is designed for immutable access `(&self)`, potentially allowing for concurrent read-only access.

```rs
/// read-only EVM database interface.
pub trait DatabaseRef {
    type Error;

    /// Get basic account information.
    fn basic_ref(&self, address: Address) -> Result<Option<AccountInfo>, Self::Error>;

    /// Get account code by its hash.
    fn code_by_hash_ref(&self, code_hash: B256) -> Result<Bytecode, Self::Error>;

    /// Get storage value of address at index.
    fn storage_ref(&self, address: Address, index: U256) -> Result<U256, Self::Error>;

    /// Get block hash by block number.
    fn block_hash_ref(&self, number: U256) -> Result<B256, Self::Error>;
}
```

In order to minimize duplication of code and the amount of interfaces to support, the `WrapDatabaseRef` struct provides a way to adapt any implementation of `DatabaseRef` to fulfill the `Database` trait. This is particularly useful when you have a read-only implementation of a database that you want to use in a context where a mutable database interface is expected. The wrapper effectively bridges the gap between these two requirements without forcing the underlying database implementation to adopt mutability.

```rs
/// Wraps a [`DatabaseRef`] to provide a [`Database`] implementation.
pub struct WrapDatabaseRef<T: DatabaseRef>(pub T);

/// Implement the wrapper around [`DatabaseRef`].
impl<F: DatabaseRef> From<F> for WrapDatabaseRef<F> {
    #[inline]
    fn from(f: F) -> Self {
        WrapDatabaseRef(f)
    }
}

/// Implement [`Database`] trait in an immutable manner.
impl<T: DatabaseRef> Database for WrapDatabaseRef<T> {
    type Error = T::Error;

    #[inline]
    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error> {
        self.0.basic_ref(address)
    }

    #[inline]
    fn code_by_hash(&mut self, code_hash: B256) -> Result<Bytecode, Self::Error> {
        self.0.code_by_hash_ref(code_hash)
    }

    #[inline]
    fn storage(&mut self, address: Address, index: U256) -> Result<U256, Self::Error> {
        self.0.storage_ref(address, index)
    }

    #[inline]
    fn block_hash(&mut self, number: U256) -> Result<B256, Self::Error> {
        self.0.block_hash_ref(number)
    }
}
```

### Gas

As for computation costs, the EVM employs a mechanism pricing mechanism called gas. In order to execute a transaction, users must pay a gas fee to compensate for the computaional resources that they spend. By doing so, we can ensure that the network is not vulnerable to spam and cannot get stuck in infinite computational loops. The gas associated with each operation is different, and must be paid regardless of the outcome of the transaction, even if it reverts.

With the introduction of [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) after the London hard fork, the fees paid by users are split between a base fee and a priority fee. This change lowered the volatility in gas prices, and gave user better predictibility.

To learn more about gas fees (structure, limits, pricing, etc.) check the following [article](https://ethereum.org/developers/docs/gas).

### Execution Environment

Environmental data necessary for the execution of the state transition. Mainly regarding the executing block, and transaction.

```rs
/// EVM environment configuration.
pub struct Env {
    /// Configuration of the block the transaction is in.
    pub block: BlockEnv,
    /// Configuration of the transaction that is being executed.
    pub tx: TxEnv,
}

/// The block environment.
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

/// The transaction environment.
pub struct TxEnv {
    /// Caller aka Author aka transaction signer.
    pub caller: Address,
    /// The gas limit of the transaction.
    pub gas_limit: u64,
    /// The gas price of the transaction.
    pub gas_price: U256,
    /// The destination of the transaction.
    pub transact_to: TransactTo,
    /// The value sent to `transact_to`.
    pub value: U256,
    /// The data of the transaction.
    pub data: Bytes,
    /// The nonce of the transaction. If set to `None`, no checks are performed.
    pub nonce: Option<u64>,
    /// The chain ID of the transaction. If set to `None`, no checks are performed.
    /// Incorporated as part of the Spurious Dragon upgrade via [EIP-155].
    pub chain_id: Option<u64>,
    /// A list of addresses and storage keys that the transaction plans to access.
    /// Added in [EIP-2930].
    pub access_list: Vec<(Address, Vec<U256>)>,
    /// The priority fee per gas.
    /// Incorporated as part of the London upgrade via [EIP-1559].
    pub gas_priority_fee: Option<U256>,
    /// The list of blob versioned hashes. Per EIP there should be at least
    /// one blob present if [`Self::max_fee_per_blob_gas`] is `Some`.
    /// Incorporated as part of the Cancun upgrade via [EIP-4844].
    pub blob_hashes: Vec<B256>,
    /// The max fee per blob gas.
    /// Incorporated as part of the Cancun upgrade via [EIP-4844].
    pub max_fee_per_blob_gas: Option<U256>,
}
```

### Interpreter

The EVM's deterministic nature ensures that Ethereum operates as a network with a state transition function. Given a current state and a series of transactions, it deterministically transitions to a new valid state.

Transactions either initiate message calls or deploy contracts. In both cases, the stack is loaded with opcodes and data (from transaction calldata, memory, or storage) to execute instructions and transition to a new state.

{{< figure src="/blog/images/revm/execution-diagram.png" align=center caption="_EVM execution model showcasing how the different components interact with each other._" >}}

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