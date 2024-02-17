---
title: "Ethereum Virtual Machine (in Rust) - Part 1"
date: 2024-02-15T19:54:00+00:00
slug: revm-pt1
category: article 
summary: Introduction to the EVM and its Rust implementation.
description: "First chapter of the Ethereum, but made in Rust series. In this first episode, I explore revm: a rust implementation of the Ethereum Virtual Machine."
cover:
  image:
  alt:
  caption: 
  relative:
showtoc: false
draft: false
---

# What are Virtual Machines?
At their core, VMs are software emulations of physical computers. They encapsulate an entire computing environment within a layer of abstraction that runs atop physical hardware. This design allows VMs to offer a sandboxed execution environment for applications, ensuring that software runs independently of the underlying hardware specifics. As you can tell, these properties are perfect for distributed systems that aim to decentralize its execution around the globe.

*But how do they work?* VMs are sophisticated programs that execute bytecode, a form of precompiled, low-level instructions designed for efficient execution by the VM. Each instruction consists of an operation code (opcode) and its arguments, guiding how the VM manipulates data and manages operations.

The execution process within a VM follows a simple yet powerful loop, with a program counter (PC) tracking its progress through the bytecode. Here’s a basic outline of this execution loop:

1. **Fetch**: Retrieve the instruction at the position indicated by the PC.
2. **Execute**: Carry out the operation specified by the instruction, which may involve arithmetic calculations, memory access, or other data manipulations.
3. **Jump**: If the instruction involves a jump, update the PC to point to the target instruction within the bytecode.
4. **Increment**: Otherwise, simply advance the PC to the next opcode. _Note that some opcodes may require inputs, and that the PC can increment by more than 1 at a time._

For instance, let's disassemble `0x6000356020350160005260206000f3` into bytes. We will assume that the opcode `0x60` consumes 1 byte, and that there are no jump instructions within this bytecode.

```md
Bytecode: 60 00 35 60 20 35 01 60 00 52 60 20 60 00 f3
      PC: 0  0  2  3  3  5  6  7  7  9  10 10 12 12 14
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

{{< figure src="/blog/images/revm/architecture.png" align=center caption="_EVM architecture and brief overview of its components._" >}}

The EVM is a stack-based machine that operates with a 1024-item-deep stack, where each item is a 256-bit word. It follows a [big endian](https://developer.mozilla.org/en-US/docs/Glossary/Endianness) byte ordering convention.

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
            #[cfg(feature = "memory_limit")]
            memory_limit: u64::MAX,
        }
    }
}
```
Since memory is a word-addressable byte array, its getter and setter methods will require an `offset` and a `size`. Note that when setting a value, its size can be derived from its length.
```rs
impl Memory {
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
Finally, as memory can be expanded up to a max capacity, the implementation should also have method to resize its buffer. Note that expansion only impacts memory size, the underlying data of the expanded section remains empty -it is filled with zeros-.
```rs
impl Memory {
    /// Resizes the memory in-place so that `len` is equal to `new_len`.
    pub fn resize(&mut self, new_size: usize) {
        self.buffer.resize(new_size, 0);
    }
}
```

### Storage

The EVM also has a **non-volatile storage** model where each account (contract) keeps relevant information of the system state. The storage layout is like a hashmap that uses **key-value pairs** to access **persistent** data. Each contract has its own storage and -at the time of writting- can only interact with their own storage.

```rs
/// An account's Storage is a mapping of 256-bit integer key-value pairs.
pub type Storage = HashMap<U256, U256>;
```
_*Note that `Hashmap` is a type defined in the standard library. As such, among other convenient methods, it already has getter and setter functions. If you are not familiar with hashmaps yet, [check its docs](https://doc.rust-lang.org/std/collections/struct.HashMap.html)._

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
    fn from(f: F) -> Self {
        WrapDatabaseRef(f)
    }
}

/// Implement [`Database`] trait in an immutable manner.
impl<T: DatabaseRef> Database for WrapDatabaseRef<T> {
    type Error = T::Error;

    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error> {
        self.0.basic_ref(address)
    }

    fn code_by_hash(&mut self, code_hash: B256) -> Result<Bytecode, Self::Error> {
        self.0.code_by_hash_ref(code_hash)
    }

    fn storage(&mut self, address: Address, index: U256) -> Result<U256, Self::Error> {
        self.0.storage_ref(address, index)
    }

    fn block_hash(&mut self, number: U256) -> Result<B256, Self::Error> {
        self.0.block_hash_ref(number)
    }
}
```


### Transient Storage

As per [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153), after the Cancun hard fork, the EVM will also implement **transient storage**. A new type of data storage mechanism that is identical to regular storage, but which is **discarded after every transaction** (only persists within a transaction). Its main application being cheaper reentrancy locks.

This difference between the two, means that the `TransientStorage` definition can be simplified. Rather than relying on a nested hashmap `State > Account [Storage] > Slot`, the key of the `TransientStorage` hashmap is a tuple composed by the account address, and the storage slot.

```rs
/// Structure used for EIP-1153 transient storage.
pub type TransientStorage = HashMap<(Address, U256), U256>;
```

### Gas

As for computation costs, the EVM employs a mechanism pricing mechanism called gas. In order to execute a transaction, users must pay a gas fee to compensate for the computaional resources that they spend. By doing so, we can ensure that the network is not vulnerable to spam and cannot get stuck in infinite computational loops. The gas associated with each operation is different, and must be paid regardless of the outcome of the transaction, even if it reverts.

Apart from the computation costs, extra gas is charged in order to form the payment for a subordinate message call or contract creation (payment is embeded in the `CREATE`, `CREATE2`, `CALL` and `CALLCODE` opcodes).

On top of that, extra gas is also charged when expanding the memory. Since memory is word-addressable, its expansion happens in 32 byte bounds. Note that expansion will happend regardless of the nature of the memory operation (either read or write).

Finally, to help address the state growth problem, gas refunds are given when storage slots are cleared.

```rs
/// Represents the state of gas during execution.
pub struct Gas {
    /// The initial gas limit.
    limit: u64,
    /// The total used gas.
    all_used_gas: u64,
    /// Used gas without memory expansion.
    used: u64,
    /// Used gas for memory expansion.
    memory: u64,
    /// Refunded gas. This is used only at the end of execution.
    refunded: i64,
}
```

With the introduction of [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) after the London hard fork, the fees paid by users are split between a base fee -common for all the txs in a block- and a user-defined priority fee. This change lowered the volatility in gas prices, and gave user better predictibility.

To learn more about gas fees (structure, limits, pricing, etc.) check the following [article](https://ethereum.org/developers/docs/gas).

### Execution Environment

Environmental data necessary for the execution of the state transition. This section is quite self-explanatory thanks to the verbose (great) comments of the `revm` contributors. Overall, it shouldn't introduce new topics to people who are already familiar with the Ethereum network.

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
    /// The calldata of the transaction.
    pub data: Bytes,
    /// The nonce of the transaction. If set to `None`, no checks are performed.
    pub nonce: Option<u64>,
    /// The chain ID of the tx. If set to `None`, no checks are performed.
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
_Note: The blob-related attributes refer to a new gas pricing mechanism for blob-carrying transactions. These are similar to regular transactions but include an additional set of data called blob (Binary Large Object). The main goal of this change is to allow roll-ups to still post their commitments on-chain. However, instead of doing it on the execution layer via calldata, they can do it on the beacon chain (consensus layer), where the blobs will be pruned in a couple of weeks. This pruning timeline gives enough time for provers to check the data commitments of the rollup and challenge them if necessary. For further detail check the [proto-danksharding website](https://www.eip4844.com/)._

`TransactTo` is an enum that specifies the target of the transaction, which can be either a call or the creation of a new contract. Note that new contracts can be deterministically created by using a user-defined salt.
```rs
pub enum TransactTo {
    /// Simple call to an address.
    Call(Address),
    /// Contract creation.
    Create(CreateScheme),
}

pub enum CreateScheme {
    /// Legacy create scheme of `CREATE`.
    Create,
    /// Create scheme of `CREATE2`.
    Create2 { salt: U256 },
}
```

## EVM Execution

The EVM's deterministic nature ensures that Ethereum operates as a network with a state transition function. Given a current state and a series of transactions, it deterministically transitions to a new valid state.

Transactions can only be triggered by externally-owned accounts (EOAs), and they either initiate message calls or deploy new contracts. In both cases, the stack is loaded with opcodes and data (from transaction calldata, memory, or storage) to execute instructions and transition to a new state.

{{< figure src="/blog/images/revm/execution-diagram.png" align=center caption="_EVM execution model showcasing how the different components interact with each other._" >}}

### The Interpreter

As seen in the previous diagram, the interpreter is the core engine of the EVM. It is in charge of running the execution loop that processes and executes each of the instructions stored in the bytecode.

Based on the above definition, we could label: program counter, stack, memory, and gas, as direct (internal) dependencies of the interpreter. Whereas storage, and the execution context, are somehow agnostic to it (external). Because of that, it seems reasonable to separate both implementations under different crates.

### The Host

This explicit separation, surfaces the need for an interface that facilitates the interaction of the EVM interpreter with its environment, encompassing essential operations such as accessing block, transaction or account data, accessing storage, or logging data. To fulfill this need, the `Host` trait has been created.

```rs
/// EVM context host.
pub trait Host {
    /// Returns a mutable reference to the environment.
    fn env(&mut self) -> &mut Env;

    /// Loads an account. Returns (is_cold, is_new_account)
    fn load_account(&mut self, address: Address) -> Option<(bool, bool)>;

    /// Get the block hash of the given block `number`.
    fn block_hash(&mut self, number: U256) -> Option<B256>;

    /// Get balance of `address` and if the account is cold.
    fn balance(&mut self, address: Address) -> Option<(U256, bool)>;

    /// Get code of `address` and if the account is cold.
    fn code(&mut self, address: Address) -> Option<(Bytecode, bool)>;

    /// Get code hash of `address` and if the account is cold.
    fn code_hash(&mut self, address: Address) -> Option<(B256, bool)>;

    /// Get storage value of `address` at `index` and if the account is cold.
    fn sload(&mut self, address: Address, index: U256) -> Option<(U256, bool)>;

    /// Set storage value of account address at index.
    /// Returns (original, present, new, is_cold).
    fn sstore(
        &mut self,
        address: Address,
        index: U256,
        value: U256,
    ) -> Option<(U256, U256, U256, bool)>;

    /// Get the transient storage value of `address` at `index`.
    fn tload(&mut self, address: Address, index: U256) -> U256;

    /// Set the transient storage value of `address` at `index`.
    fn tstore(&mut self, address: Address, index: U256, value: U256);

    /// Emit a log owned by `address` with given `LogData`.
    fn log(&mut self, log: Log);

    /// Mark `address` to be deleted, with funds transferred to `target`.
    fn selfdestruct(
        &mut self,
        address: Address,
        target: Address
    ) -> Option<SelfDestructResult>;
}
```

This abstraction allows the interpreter to interact with any host environment -as long as the trait is implemented-, thereby enhancing modularity and interoperability. Different implementations can be used to simulate different environments when connecting to different EVM-compatible networks.

### A closing example

In the next article of the series, we will review in detail how each opcode works. On the meantime, in order to further exemplify how the execution of an EVM transaction works, the following snippet showcases a representation of the bytecode that we initially disassembled: `0x6000356020350160005260206000f3`.

```
PC   BYTECODE   MNEMONIC       STACK          ACTION
0    60 00      PUSH1 0x00     [0x00]         Push 0 to the stack.
2    35         CALLDATALOAD   [num1]         Load first 32 bytes on the stack.
3    60 20      PUSH1 0x20     [0x20, num1]   Push 32 to the stack.
5    35         CALLDATALOAD   [num2, num1]   Load second 32 bytes on the stack.
6    01         ADD            [n2+n1]        Take 2 stack inputs and add them.
7    60 00      PUSH1 0x00     [0x0, n2+n1]   Push 0 to the stack.
9    52         MSTORE         []             Store n2+n1 with offset 0 in memory.
10   60 20      PUSH1 0x20     [0x20]         Push 32 to the stack.
12   60 00      PUSH1 0x00     [0x00, 0x20]   Push 0 to the stack.
14   f3         RETURN         []             Return the first 32 bytes of memory.
```

Given 64 bytes of calldata (2 words), and by executing our small bytecode with the EVM interpreter, we loaded them into the stack, added them, stored them in memory, and finally returned the output. Quite an achievement!

---

That's a wrap! In this article, we've discussed the basics of the EVM, its high-level mechanics, and its core building blocks. Armed with this knowledge, we are better prepared to understand how these components interact with each other.

Moving forward, in the next article, we will delve into the EVM interpreter, exploring its role and functionality in greater detail. Specifically, we will focus on the instruction set that powers the EVM.