---
title: "Ethereum Virtual Machine (in Rust) - Part 2"
date: 2024-03-04T16:00:00+00:00
slug: revm-pt2
category: article 
summary: A closer look to the revm interpreter.
description: "In this second article of the revm series, we will dive deeper into the EVM interpreter. We will understand how it works, especially focusing on the different opcodes."
cover:
  image:
  alt:
  caption: 
  relative:
showtoc: true
draft: false
---

# Interpreter

As uncovered in the [previous article](../revm-pt1) —which I highly recommend checking out if you haven't yet—, the interpreter is the core engine of the EVM. It is in charge of running the execution loop that processes and executes each of the instructions stored in the bytecode. Additionally, besides the computation and returning the results of the execution, the interpreter also needs to keep track of the gas consumption.

To fulfill its tasks, the interpreter will use its attributes (contract bytecode, stack, memory, etc.) and also interact with the EVM host.

{{< figure src="/blog/images/revm/interpreter-diagram.png" align=center caption="_Representation of revm's interpreter._" >}}

The following definition of the `Interpreter` struct gives it control over all the attributes described in the diagram above. Additionally, the `InstructionResult` enum, and the `Contract` struct, are also defined.

```rs
pub struct Interpreter {
    // Contract information and invoking data
    pub contract: Box<Contract>,
    // The current instruction pointer.
    pub instruction_pointer: *const u8,
    // Execution control flag. If not set to `Continue`, the execution will stop.
    pub instruction_result: InstructionResult,
    // The gas state.
    pub gas: Gas,
    // Memory. 
    // Only set while running the loop. Otherwise taken and emptied.
    pub memory: Memory,
    // Stack.
    pub stack: Stack
    // When the execution finished, contains the output bytes of the contract.
    pub return_data: Bytes,
}
```
The `InstructionResult` enum categorizes the outcomes of instruction executions, providing granularity and enabling the interpreter to handle success, reverts, and errors elegantly. These components, among others, ensure the interpreter's smooth operation and robust error handling.

```rs
pub enum InstructionResult {
    // Success
    #[default]
    Continue = 0x00, Stop, Return,
    // Revert
    Revert = 0x10, CallTooDeep, OutOfFunds,
    // Errors
    OutOfGas = 0x50, OpcodeNotFound, StackUnderflow, StackOverflow, // ...
}

impl InstructionResult {
    // Returns whether the result is a success.
    pub const fn is_ok(self) -> bool {
        matches!(self, crate::return_ok!())
    }

    // Returns whether the result is a revert.
    pub const fn is_revert(self) -> bool {
        matches!(self, crate::return_revert!())
    }

    // Returns whether the result is an error.
    pub const fn is_error(self) -> bool {
        matches!(self, return_error!())
    }
}
```
The usage of macros is just for maintainability purposes, as they simply return the enum variants that match the outcome of the match statement. When new reason codes are added to the EVM, code maintainers simply need to add a new variant to the macro.
```rs
macro_rules! return_ok {
    () => {
        InstructionResult::Continue
            | InstructionResult::Stop
            | InstructionResult::Return
    };
}
```

## Contracts

The `Contract` struct holds all data related to the target bytecode to be executed, as well as the invoking data in its function call.

```rs
// EVM contract information.
pub struct Contract {
    // Contracts data
    pub input: Bytes,
    // Contract code, size of original code, and jump table.
    // Note that current code is extended with push padding and STOP at the end.
    pub bytecode: BytecodeLocked,
    // Bytecode hash.
    pub hash: B256,
    // Contract address
    pub address: Address,
    // Caller of the EVM.
    pub caller: Address,
    // Value send to contract.
    pub value: U256,
}

```
Before exploring `BytecodeLocked`, let's examine the initialization functions for the `Contract` struct, including the standard `fn new()` method and the convenient `fn new_env()` to create a new contract instance from the current EVM environment.

```rs
impl Contract {
    // Instantiates a new contract by analyzing the given bytecode.
    pub fn new(
            input: Bytes,
            bytecode: Bytecode,
            hash: B256,
            address: Address,
            caller: Address,
            value: U256,
        ) -> Self {
            let bytecode = to_analysed(bytecode).try_into().expect("analysed");
            Self {input, bytecode, hash, address, caller, value}
        }

    // Creates a new contract from the given [`Env`].
    pub fn new_env(env: &Env, bytecode: Bytecode, hash: B256) -> Self {
        let address = match env.tx.transact_to {
            TransactTo::Call(caller) => caller,
            TransactTo::Create(..) => Address::ZERO,
        };
        let data = env.tx.data.clone();
        Self::new(data, bytecode, hash, address, env.tx.caller, env.tx.value)
    }
}
```
### Bytecode: type definition and analysis

When we explored the building blocks of the EVM, we simply mentioned that the world state holds all the basic account information, including its bytecode. Nevertheless, there is more to it.

Later on, we will analyze in depth how do push and jump-related opcodes work, but for the time being you should know that:
- `PUSH` opcodes are characterized for consuming bytes directly from the contract bytecode, to use as inputs for the interpreter.
- `JUMP`, and `JUMPI` opcodes are a control-flow-type of opcodes that require a pre-defined valid jump destination `JUMPDEST` to be informed within the bytecode.

Given the complex interplay between push and jump-related opcodes, verifying jump destinations during execution becomes non-trivial and inefficient. This complexity arises due to the need to differentiate between genuine jump destinations those potentially disguised as inputs from `PUSH` opcodes. By performing a pre-execution analysis of the bytecode to identify all valid jump destinations, a `JumpMap` can be created beforehand. This proactive approach streamlines execution and eliminates the inefficiency of runtime checks.

```rs
// A vector where each element represents a byte from the contract code.
// Each element is marked as either an invalid (0) or valid (1) jump destination.
pub struct JumpMap(pub Arc<BitVec<u8>>);

impl JumpMap {
    // Check if `pc` is a valid jump destination.
    pub fn is_valid(&self, pc: usize) -> bool {
        pc < self.0.len() && self.0[pc]
    }
}

// Analyze bytecode to build a jump map.
fn analyze(code: &[u8]) -> JumpMap {
    let mut jumps: BitVec<u8> = bitvec![u8, Lsb0; 0; code.len()];

    let range = code.as_ptr_range();
    let start = range.start;
    let mut iter = start;
    let end = range.end;
    while iter < end {
        let opcode = unsafe { *iter };
        if opcode::JUMPDEST == opcode {
            // SAFETY: jumps are max length of the code
            unsafe { jumps.set_unchecked(iter.offset_from(start) as usize, true) }
            iter = unsafe { iter.offset(1) };
        } else {
            let push_offset = opcode.wrapping_sub(opcode::PUSH1);
            if push_offset < 32 {
                // SAFETY: iterator access range is checked in the while loop
                iter = unsafe { iter.offset((push_offset + 2) as isize) };
            } else {
                // SAFETY: iterator access range is checked in the while loop
                iter = unsafe { iter.offset(1) };
            }
        }
    }

    JumpMap(Arc::new(jumps))
}
```
As analyzing the bytecode requires some computation, for the sake of performance, it is useful to define a `Bytecode` struct with different `BytecodeState`. By defining 3 different states, a more granular and efficient execution model is achieved. The transition from `Raw` to `Checked` is used to pad the bytecode with multiples of 33, ensuring that the PC can be increased by 1 in any given position of the initial bytecode, and that the last opcode of the padded code will be `0x00 (STOP)`. Finally, bytecode that has been `Checked` will then undergo analysis and transition to `Analysed`.

```rs
pub struct Bytecode {
    pub bytecode: Bytes,
    pub state: BytecodeState,
}

pub enum BytecodeState {
    // No analysis has been performed.
    Raw,
    // The bytecode has been checked for validity.
    Checked { len: usize },
    // The bytecode has been analysed for valid jump destinations.
    Analysed { len: usize, jump_map: JumpMap },
}

// Pads the bytecode and ensures 0x00 (STOP) is the last opcode.
pub fn to_checked(self) -> Self {
    match self.state {
        BytecodeState::Raw => {
            let len = self.bytecode.len();
            let mut padded_bytecode = Vec::with_capacity(len + 33);
            padded_bytecode.extend_from_slice(&self.bytecode);
            padded_bytecode.resize(len + 33, 0);
            Self {
                bytecode: padded_bytecode.into(),
                state: BytecodeState::Checked { len },
            }
        }
        _ => self,
    }
}

// Finds and caches valid jump destinations for later execution.
pub fn to_analysed(bytecode: Bytecode) -> Bytecode {
    let (bytecode, len) = match bytecode.state {
        BytecodeState::Raw => {
            let len = bytecode.bytecode.len();
            let checked = bytecode.to_checked();
            (checked.bytecode, len)
        }
        BytecodeState::Checked { len } => (bytecode.bytecode, len),
        _ => return bytecode,
    };
    let jump_map = analyze(bytecode.as_ref());

    Bytecode {
        bytecode,
        state: BytecodeState::Analysed { len, jump_map },
    }
}
```
 Additionally, analysed bytecode can be converted into `BytecodeLocked`, signaling it has passed all checks and analyses, thereby locking it against further modifications. Then, it will be feedable to the interpreter's `Contract`.

```rs
// Analysed bytecode.
pub struct BytecodeLocked {
    bytecode: Bytes,
    original_len: usize,
    jump_map: JumpMap,
}

impl TryFrom<Bytecode> for BytecodeLocked {
    type Error = ();

    fn try_from(bytecode: Bytecode) -> Result<Self, Self::Error> {
        if let BytecodeState::Analysed { len, jump_map } = bytecode.state {
            Ok(BytecodeLocked {
                bytecode: bytecode.bytecode,
                original_len: len,
                jump_map,
            })
        } else {
            Err(())
        }
    }
}
```

## Opcodes

As you may have already anticipated, efficient methods for associating opcodes with their corresponding execution logic and mnemonic representation are essential. `revm` achieves it thanks to the creation of an `InstructionTable` and an `OPCODE_JUMPMAP`.

### Basics Types: `OpCode` and `Instruction`

The `Instruction` type is a function pointer with a specific signature that represents the implementation of an EVM opcode. This function takes two arguments: a mutable reference to the `Interpreter` and a mutable reference to the `Host`. Similarly to the opcode jumpmap, in this case it is also convenient to define `InstructionTable`, an array of instruction pointers.

```rs
// EVM opcode function signature.
pub type Instruction<H> = fn(&mut Interpreter, &mut H);

// List of instruction function pointers mapped to the 256 EVM opcodes.
pub type InstructionTable<H> = [Instruction<H>; 256];
```

By wrapping a single byte, the `OpCode` struct represents an individual EVM opcode in a human-readable, type-safe manner. Each byte corresponds to one of the 256 possible values defined by the `OPCODE_JUMPMAP`, ensuring that opcodes are only instantiated with valid values, as not all possible values are associated with an opcode.

The `OPCODE_JUMPMAP` is a static jump map array that links opcode values to their mnemonic representation. It is useful for debugging and introspection, helping devs print or log opcode names rather than hexadecimal values.

```rs
// An EVM opcode. Always valid, as it is declared in the [`OPCODE_JUMPMAP`].
pub struct OpCode(u8);

impl OpCode {
    // Instantiate a new opcode from a u8.
    pub const fn new(opcode: u8) -> Option<Self> {
        match OPCODE_JUMPMAP[opcode as usize] {
            Some(_) => Some(Self(opcode)),
            None => None,
        }
    }
}
```
After defining the basic custom types related to opcodes, the next step involves the implementation of a mechanism that not only generates but also seamlessly links these types with their corresponding execution logic. This is where the `opcodes!` macro comes into play. This macro that takes a series of inputs separated by commas, where each input refers to an opcode and. At the same time, each input has 3 sub-inputs separated by arrows `=>`:
- `$val:literal`: opcode value as a hexadecimal number.
- `$name:ident`: mnemonic representation of the opcode.
- `$f:expr`: expression that resolves to the function implementing the instruction logic.

The macro then produces several outputs:
- **Opcode Constants**: For each opcode provided to the macro, a constant is declared with the given name `$name` and hexadecimal value `$val`. This allows the opcodes to be referenced by name throughout the codebase.
- **Opcode Jumpmap**: As previously described, the macro creates a static jump map array that maps opcode values to their mnemonic representations. The `OPCODE_JUMPMAP` array is initially filled with `None` values. Then, the corresponding entry for each opcode is set to `Some(stringify!($name))`, turning the mnemonic into a string.
- **Instruction Function Dispatcher**: Finally, the macro generates a dispatcher mechanism; `fn instruction()` that takes an opcode value, and an EVM spec, and returns the corresponding `Instruction` execution logic. A match statement to map each opcode value `$val` to its function `$f` is used. If an unknown opcode is encountered, the macro defaults to `control::unknown` to handle undefined opcodes.

```rs
macro_rules! opcodes {
    ($($val:literal => $name:ident => $f:expr),* $(,)?) => {
        // Constants for each opcode. This also takes care of duplicate names.
        $(
            pub const $name: u8 = $val;
        )*

        // Maps each opcode to its name.
        pub const OPCODE_JUMPMAP: [Option<&'static str>; 256] = {
            let mut map = [None; 256];
            let mut prev: u8 = 0;
            $(
                let val: u8 = $val;
                assert!(val == 0 || val > prev, "opcodes must be in asc order");
                prev = val;
                map[$val] = Some(stringify!($name));
            )*
            let _ = prev;
            map
        };

        // Returns the instruction function for the given opcode and spec.
        pub fn instruction<H: Host, SPEC: Spec>(opcode: u8) -> Instruction<H> {
            match opcode {
                $($name => $f,)*
                _ => control::unknown,
            }
        }
    };
}
```
The `InstructionTable` is typically generated by iterating over all possible opcode values (from 0 to 255) and calling `fn instruction()` with each of them. This process effectively builds an array where each index corresponds to an opcode, and where the value at that index is used as its `Instruction` function pointer. By defining the `InstructionTable` in such a way, every possible opcode is guaranteed to have an associated function -or, at least, a the default undefined implementation- in the table.

Since the `InstructionTable` plays a key role in the interpreter's execution, the benefits of this design aren't limited to convenience; using a fixed-sized table (indexable by opcode value) allows for constant-time lookup of instruction functions. Rather than iterating through enums or using match/case statements with potentially slower lookup times, this design provides a more performant approach.

`revm` also implements a `Boxed` version of both types. By doing so, devs who want to implement their own even, can have closures available for `BoxedInstruction` at the expense of some performance. Because of that, both implementations are wrapped under an enum.

```rs
pub enum InstructionTables<'a, H: Host> {
    // `Plain` gives 10-20% faster Interpreter execution.
    Plain(InstructionTable<H>),
    // `Boxed` wraps the plain function pointers with closures.
    Boxed(BoxedInstructionTable<'a, H>),
}
```

The following snippet showcases how the `opcodes!` macro is called to instantiate the `OPCODE_JUMPMAP` with the map between opcode hexadecimal values and their mnemonic representation, as well as linking the `fn instruction()` implementation to each opcode value. [Check the full code here](https://github.com/bluealloy/revm/blob/93c7ba0ff619534751df2922e7e77671b79077ff/crates/interpreter/src/instructions/opcode.rs#L94-#L357).

```rs
opcodes! {
    0x00 => STOP   => control::stop,
    0x01 => ADD    => arithmetic::wrapping_add,
    0x02 => MUL    => arithmetic::wrapping_mul,
    // ...
    0x60 => PUSH1  => stack::push::<1, H>,
    0x61 => PUSH2  => stack::push::<2, H>,
    // ...
    0x50 => POP    => stack::pop,
    0x51 => MLOAD  => memory::mload,
    0x52 => MSTORE => memory::mstore,
    // ...
    0xF3 => RETURN => control::ret,
}
```

Perhaps the benefits were not obvious at first, yet it should now be clear that by using a macro:
- adding opcodes becomes a matter of adding entries to its invocation.
- updating the logic of an opcode is a matter of redefining the function linked to its instruction function pointer.

Rather than manually updating enums, or match statements, this approach establishes a separation of concerns; making the codebase less error-prone and easier to understand, test, and extend -as the opcode execution logic can evolve independently of how opcodes are defined and organized-.

### More types: `OpInfo`

The `OpInfo` struct also plays an important role in opcode handling, providing a compact and efficient way to store and access relevant information for the execution of each opcode. By packing this data into a single u32, `OpInfo` minimizes memory usage while maintaining quick access to opcode metadata.

```rs
pub struct OpInfo {
    /// Data contains few information packed inside u32:
    /// IS_JUMP (1bit) | IS_GAS_BLOCK_END (1bit) | IS_PUSH (1bit) | GAS (29bits)
    data: u32,
}
```

- `IS_JUMP` **(1 bit):** Indicates if the opcode is a jump operation (`JUMP` or `JUMPI`). This information is useful for control flow management.
- `IS_GAS_BLOCK_END` **(1 bit):** Marks the opcode as ending a block of operations that are considered for gas calculation purposes.
- `IS_PUSH` **(1 bit):** Identifies the opcode as a push operation.
- `GAS` **(29 bits)**: Stores the gas cost associated with the opcode. This allows for quick access to gas requirements, facilitating efficient gas calculation during execution.
    
As the information is packed in a single field, to manipulate and interpret the `data` within `OpInfo`, several bitmasks and utility functions are defined:

```rs
const JUMP_MASK: u32 = 0x80000000;
const GAS_BLOCK_END_MASK: u32 = 0x40000000;
const IS_PUSH_MASK: u32 = 0x20000000;
const GAS_COST_MASK: u32 = 0x1FFFFFFF;
```

_Note: We will dive deeper into gas later on the series but, as a rule of thumb, you can assume that all opcodes have a fixed gas cost: `ZERO`, `BASE`, `VERYLOW`, `LOW`, `MID`, `HIGH`, or custom values. Additionally, some opcodes will also incur a dynamic gas costs depending on their runtime behavior._

Most of the `OpInfo` elements are instantiated by only informing their gas cost (as they are neither jumps, push, or end-of-block opcodes) using `fn gas()`. Additionally, to get the gas cost associated with an opcode, we simply need to apply the `GAS_COST_MASK` to the data by using a bitwise `AND` operator.

```rs
impl OpInfo {
    /// Creates a new [`OpInfo`] with the given gas value.
    pub const fn gas(gas: u64) -> Self {
        Self { data: gas as u32 }
    }

    /// Returns the gas cost of the opcode.
    pub fn get_gas(self) -> u32 {
        self.data & GAS_MASK
    }
}
```

For each bitmask, corresponding generator and checker functions are implemented. For the sake of brevity, we will only look at the ones related to push opcodes. On the one hand, we have the constructor `fn push_opcode()`, which stores the gas information in the first 29 bits, and uses a bitwise `OR` in conjunction with the `IS_PUSH_MASK` to flip the bit that sets the opcode to `IS_PUSH`. On the other hand, to check whether an opcode `IS_PUSH`, we simply need to use the `IS_PUSH_MASK` to check if the relevant bit is set to 1.

```rs
impl OpInfo {
    /// Creates a new push [`OpInfo`].
    pub const fn push_opcode() -> Self {
        Self {
            data: gas::VERYLOW as u32 | IS_PUSH_MASK,
        }
    }

    /// Whether the opcode is a PUSH opcode or not.
    pub fn is_push(self) -> bool {
        self.data & IS_PUSH_MASK == IS_PUSH_MASK
    }
}
```

Finally, similar to `InstructionTable` or `OPCODE_JUMPMAP`, it is useful to create a table (a fixed-size array indexable by opcode value) with the generated `OpInfo`, to increase runtime performance.

```rs
const fn make_gas_table(spec: SpecId) -> [OpInfo; 256] {
    let mut table = [OpInfo::none(); 256];
    let mut i = 0;
    while i < 256 {
        table[i] = opcode_gas_info(i as u8, spec);
        i += 1;
    }
    table
}

const fn opcode_gas_info(opcode: u8, spec: SpecId) -> OpInfo {
    match opcode {
        STOP => OpInfo::gas_block_end(0),
        ADD => OpInfo::gas(gas::VERYLOW),
        MUL => OpInfo::gas(gas::LOW),
        // ...
        PUSH1 => OpInfo::push_opcode(),
        PUSH2 => OpInfo::push_opcode(),
        // ...
        POP => OpInfo::gas(gas::BASE),
        MLOAD => OpInfo::gas(gas::VERYLOW),
        MSTORE => OpInfo::gas(gas::VERYLOW),
        // ...
        RETURN => OpInfo::gas_block_end(0),
    }
}
```
Note that besides maintaining the `opcodes!` macro, `opcode_gas_info` must also be maintained for gas updates.

## Logic Implementation

After analyzing all the building blocks, it is now time to dive into the implementation of each opcode. First though, let's see how the interpreter consumes and processes each opcode in the execution loop.

```rs
impl Interpreter {
    // Executes the instruction at the current instruction pointer.
    // Internally it will increment instruction pointer by one.
    fn step<FN, H: Host>(&mut self, instruction_table: &[FN; 256], host: &mut H)
    where
        FN: Fn(&mut Interpreter, &mut H),
    {
        // Get current opcode (hexadecimal value).
        let opcode = unsafe { *self.instruction_pointer };

        // SAFETY: In analysis > padding of bytecode to ensure last byte is STOP.
        // Therefore, on last instruction it will stop execution of this contract.
        self.instruction_pointer = unsafe { self.instruction_pointer.offset(1) };

        // Execute instruction by using the [`InstructionTable`].
        (instruction_table[opcode as usize])(self, host)
    }
}
```

Thanks to the `fn step()` function, as previously described, the interpreter loop processes one opcode at a time. It does so by executing the corresponging `Instruction` form the `InstructionTable` based on the processed opcode value.

### Useful Macros

Before diving into the actual opcode implementations, we will first define some useful macros that are frequently used by the instructions. Other than for convenience purposes, these macros are generally used as a workaround to set interpreter results and return in case of an error, as the opcode `Instruction` does not return any values.

The `gas!` macro updates the acrued gas cost of the current interpreter execution. It does so by calling `fn record_gas()` on the gas attribute of the interpreter. If the call execution runs out of gas, it will set the `InstructionResult::OutOfGas`.

```rs
macro_rules! gas {
    ($interp:expr, $gas:expr) => {
        if !$interp.gas.record_cost($gas) {
            $interp.instruction_result = InstructionResult::OutOfGas;
            return;
        }
    };
}
```

As the `Gas` struct [was only presented](../revm-pt1/#gas) but its implementation wasn't shown, let's have a look now. As the snippet below shows, `fn record_cost()` increases the gas used, whereas `fn record_memory()` increases the memory-related gas costs. In both cases, the total gas used is updated, and the returned value is `false` if the gas limit is exceeded.

```rs
impl Gas {
    /// Records an explicit cost. Returns `false` if gas limit is exceeded.
    pub fn record_cost(&mut self, cost: u64) -> bool {
        let all_used_gas = self.all_used_gas.saturating_add(cost);
        if self.limit < all_used_gas {
            return false;
        }

        self.used += cost;
        self.all_used_gas = all_used_gas;
        true
    }

    /// Records memory expansion cost. Returns `false` if gas limit is exceeded.
    pub fn record_memory(&mut self, gas_memory: u64) -> bool {
        if gas_memory > self.memory {
            let all_used_gas = self.used.saturating_add(gas_memory);
            if self.limit < all_used_gas {
                return false;
            }
            self.memory = gas_memory;
            self.all_used_gas = all_used_gas;
        }
        true
    }
}
```

The `pop!` macro is a flexible procedural macro that pops up to 4 words (maximum popped by any opcode) from the stack, while handling reverts in the event of a `InstructionResult::StackUnderflow`. Similarly, `pop_address!` and `pop_top!` are also implemented. While `pop_address!` converts the resulting `U256` words into `Address`, `pop_top!` pops up to 2 words and returns a reference to the new top-most item of the stack.

```rs
macro_rules! pop {
    ($interp:expr, $x1:ident) => {
        if $interp.stack.len() < 1 {
            $interp.instruction_result = InstructionResult::StackUnderflow;
            return;
        }
        // SAFETY: Length is checked above.
        let $x1 = unsafe { $interp.stack.pop_unsafe() };
    };
    ($interp:expr, $x1:ident, $x2:ident) => {
        if $interp.stack.len() < 2 {
            $interp.instruction_result = InstructionResult::StackUnderflow;
            return;
        }
        // SAFETY: Length is checked above.
        let ($x1, $x2) = unsafe { $interp.stack.pop2_unsafe() };
    };
    ($interp:expr, $x1:ident, $x2:ident, $x3:ident) => {
        if $interp.stack.len() < 3 {
            $interp.instruction_result = InstructionResult::StackUnderflow;
            return;
        }
        // SAFETY: Length is checked above.
        let ($x1, $x2, $x3) = unsafe { $interp.stack.pop3_unsafe() };
    };
    ($interp:expr, $x1:ident, $x2:ident, $x3:ident, $x4:ident) => {
        if $interp.stack.len() < 4 {
            $interp.instruction_result = InstructionResult::StackUnderflow;
            return;
        }
        // SAFETY: Length is checked above.
        let ($x1, $x2, $x3, $x4) = unsafe { $interp.stack.pop4_unsafe() };
    };
}
```

The `push!` and `push_b256` macro simply push a new word (`U256` or `B256`) into the stack. As previously explained, the reason behind its usage is the ability to set the interpreter's `InstructionResult::StackOverflow` and return in case of an error (which is not possible in the instruction function definition).
```rs
macro_rules! push {
    ($interp:expr, $($x:expr),* $(,)?) => ($(
        match $interp.stack.push($x) {
            Ok(()) => {},
            Err(e) => {
                $interp.instruction_result = e;
                return;
            }
        }
    )*)
}
```

The `memory_resize!` macro expands the memory if necessary. If the memory limit is reached, it will set the `InstructionResult::MemoryLimitOOG`. Additionally, it will calculate the extra gas of the memory expansion costs.

```rs
macro_rules! memory_resize {
    ($interp:expr, $offset:expr, $len:expr) => {
        memory_resize!($interp, $offset, $len, ())
    };
    ($interp:expr, $offset:expr, $len:expr, $ret:expr) => {
        let size = $offset.saturating_add($len);
        if size > $interp.memory.len() {
            if $interp.memory.limit_reached(size) {
                $interp.instruction_result = InstructionResult::MemoryLimitOOG;
                return $ret;
            }

            // We are fine with saturating to usize if size is close to MAX value.
            let rounded_size = crate::interpreter::next_multiple_of_32(size);

            // Gas is calculated in EVM words (256bits).
            let words_num = rounded_size / 32;
            if !$interp.gas.record_memory(crate::gas::memory_gas(words_num)) {
                $interp.instruction_result = InstructionResult::MemoryLimitOOG;
                return $ret;
            }

            // Gas of memory expansion cost (not covered yet)
            $interp.memory.resize(rounded_size);
        }
    };
}
```

If the interpreter is supposed to be in view-only mode, the `check_staticcall!` macro will set the `InstructionResult::StateChangeStaticCall` and return.

```rs
macro_rules! check_staticcall {
    ($interp:expr) => {
        if $interp.is_static {
            $interp.instruction_result = InstructionResult::StateChangeStaticCall;
            return;
        }
    };
}
```

Finally, `as_u64_saturated!` and `as_usize_or_fail!` macros are useful to convert inputs to `u64` (saturating to `u64::MAX` rather than overflowing) or `usize` (reverting if not possible).

---

Now that the useful interpreter macros are understood, the following sections will delve into the implementation functions for various opcode instructions. In addition to the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf), which provides a highly mathematical perspective, [evm.codes](https://www.evm.codes) serves as a practical, and therefore, extremely useful resource. It offers not only comprehensive and clear explanations but also features a playground to thoroughly explore the behavior of arbitrary (user-input) bytecode and calldata.

_Note: Instructions can be groupped in several ways. This article will follow `revm`'s groupping criteria which is similar yellow paper, but more granular._

### Arithmetic Instructions

The EVM has several opcodes to perform arithmetic operations based on different inputs previously loaded onto the stack.

```rs
// Snippet of the `opcodes!` macro to showcase EVM's arithmetic opcodes,
// their mnemonic representation, and their [`Instruction`].
opcodes! {
    0x01 => ADD        => arithmetic::wrapping_add,
    0x02 => MUL        => arithmetic::wrapping_mul,
    0x03 => SUB        => arithmetic::wrapping_sub,
    0x04 => DIV        => arithmetic::div,
    0x05 => SDIV       => arithmetic::sdiv,
    0x06 => MOD        => arithmetic::rem,
    0x07 => SMOD       => arithmetic::smod,
    0x08 => ADDMOD     => arithmetic::addmod,
    0x09 => MULMOD     => arithmetic::mulmod,
    0x0A => EXP        => arithmetic::exp::<H, SPEC>,
    0x0B => SIGNEXTEND => arithmetic::signextend,
}
```

As explained in the [previous article](../revm-pt1), `revm` is built on [ruint](https://github.com/recmo/uint) primitive types. Thanks to this foundation, the implementation of arithmetic instructions is trivial, as it relies on the built-in methods of the `U256` type defined in the imported crate. Because of this, we will focus solely on the implementation of the `ADD` opcode.

```rs
pub fn wrapping_add<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop_top!(interpreter, op1, op2);
    *op2 = op1.wrapping_add(*op2);
}
```

As previously mentioned in the [useful macros section](./#useful-macros), the `gas!` macro will record the gas cost associated with the computation of the instruction. Then, the `pop_top!` macro pops up to 2 words and returns a reference to the new top-most item on the stack. `fn wrapping_add()` leverages this macro to seamlessly dereference the second stack input and update its value with the result of the arithmetic operation. Note that this process is equivalent to popping the words from the stack, adding them together, and then pushing the result back onto the stack. Nevertheless, by using the `pop_top!` macro, this operation can be executed more concisely and efficiently.

### Bitwise Instructions

The EVM has several opcodes to perform bitwise operations based on different inputs previously loaded onto the stack.

```rs
// Snippet of the `opcodes!` macro to showcase EVM's bitwise opcodes,
// their mnemonic representation, and their [`Instruction`].
opcodes! {
    0x10 => LT     => bitwise::lt,
    0x11 => GT     => bitwise::gt,
    0x12 => SLT    => bitwise::slt,
    0x13 => SGT    => bitwise::sgt,
    0x14 => EQ     => bitwise::eq,
    0x15 => ISZERO => bitwise::iszero,
    0x16 => AND    => bitwise::bitand,
    0x17 => OR     => bitwise::bitor,
    0x18 => XOR    => bitwise::bitxor,
    0x19 => NOT    => bitwise::not,
    0x1A => BYTE   => bitwise::byte,
    0x1B => SHL    => bitwise::shl::<H, SPEC>,
    0x1C => SHR    => bitwise::shr::<H, SPEC>,
    0x1D => SAR    => bitwise::sar::<H, SPEC>,
}
```

This block of instructions is also trivial to implement thanks to the built-in methods of the `U256` type. Bitwise operations can be split into two categories: bit-comparison operations, which return `true` or `false`, and bit-modifying operations. However, since the EVM uses U256 for all values, these boolean outcomes are represented as `U256::from(1)` (true) or `U256::from(0)` (false).

```rs
pub fn eq<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop_top!(interpreter, op1, op2);
    *op2 = U256::from(op1 == *op2);
}

pub fn bitand<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop_top!(interpreter, op1, op2);
    *op2 = op1 & *op2;
}
```

Bitwise operations follow the same approach as [arithmetic operations](./#arithmetic-instructions), using the `gas!` and `pop_top!` macros to update the gas cost associated with the ongoing interpreter execution and to modify the stack accordingly.

### Environment Instructions

Environment instructions refer to those operations that are related to information stored on the `Env` struct, which the interpreter can access via the `fn env()` method of the `Host` trait.

```rs
// Snippet of the `opcodes!` macro to showcase environment opcodes,
// their mnemonic representation, and their [`Instruction`].
opcodes! {
    0x32 => ORIGIN      => host_env::origin,
    0x3A => GASPRICE    => host_env::gasprice,
    0x41 => COINBASE    => host_env::coinbase,
    0x42 => TIMESTAMP   => host_env::timestamp,
    0x43 => NUMBER      => host_env::number,
    0x44 => DIFFICULTY  => host_env::difficulty::<H, SPEC>,
    0x45 => GASLIMIT    => host_env::gaslimit,
    0x46 => CHAINID     => host_env::chainid::<H, SPEC>,
    0x48 => BASEFEE     => host_env::basefee::<H, SPEC>,
    0x49 => BLOBHASH    => host_env::blob_hash::<H, SPEC>,
    0x4A => BLOBBASEFEE => host_env::blob_basefee::<H, SPEC>,
}
```

This block of instructions is also trivial to implement, as it simply pushes new environmental information (which is easily accessible thanks to the `Host` trait) onto the stack. The instructions simply call the `gas!` and `push!` macros to update the gas cost associated with the ongoing interpreter execution and to modify its stack accordingly.

```rs
pub fn origin<H: Host>(interpreter: &mut Interpreter, host: &mut H) {
    gas!(interpreter, gas::BASE);
    push_b256!(interpreter, host.env().tx.caller.into_word());
}

pub fn timestamp<H: Host>(interpreter: &mut Interpreter, host: &mut H) {
    gas!(interpreter, gas::BASE);
    push!(interpreter, host.env().block.timestamp);
}
```

### System Instructions

System instructions refer to operations related to information in the `Interpreter` struct, excluding  `Memory` and `Stack` operations, which have their own set of opcodes.

Instructions `0x30..0x39` relate to `Contract` information, whereas instruction `0x5A` relates to `Gas` costs.

```rs
// Snippet of the `opcodes!` macro to showcase EVM's system opcodes,
// their mnemonic representation, and their [`Instruction`].
opcodes! {
    0x30 => ADDRESS        => system::address,
    0x33 => CALLER         => system::caller,
    0x34 => CALLVALUE      => system::callvalue,
    0x35 => CALLDATALOAD   => system::calldataload,
    0x36 => CALLDATASIZE   => system::calldatasize,
    0x37 => CALLDATACOPY   => system::calldatacopy,
    0x38 => CODESIZE       => system::codesize,
    0x39 => CODECOPY       => system::codecopy,
    0x5A => GAS            => system::gas,
}
```

Firstly, the simplest opcodes will simply push new information into the stack.

```rs
// Push the address of the [`Contract`] address onto the stack.
pub fn address<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    push_b256!(interpreter, interpreter.contract.address.into_word());
}

// Push the address of the [`Contract`] caller onto the stack.
pub fn caller<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    push_b256!(interpreter, interpreter.contract.caller.into_word());
}

// Push the value of the [`Contract`] call onto the stack.
pub fn callvalue<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    push!(interpreter, interpreter.contract.value);
}

// Push the size of the [`Contract`] calldata onto the stack.
pub fn calldatasize<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    push!(interpreter, U256::from(interpreter.contract.input.len()));
}

// Push the size of the [`Contract`] code onto the stack.
pub fn codesize<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    push!(interpreter, U256::from(interpreter.contract.bytecode.len()));
}

// Push the remaining [`Gas`] value onto the stack.
pub fn gas<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    push!(interpreter, U256::from(interpreter.gas.remaining()));
}
```

Secondly, `CALLDATALOAD` opcodes will take an input from the stack (offset) and 32-bytes of calldata (starting from the offset) into the stack. If the offset is bigger than the calldata length, a zero will be pushed.

```rs
// Push 1 word of call data into the stack. Load from `index` (offfset).
pub fn calldataload<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop!(interpreter, index);
    let index = as_usize_saturated!(index);
    let load = if index < interpreter.contract.input.len() {
        let length = 32.min(interpreter.contract.input.len() - index);
        let end = index + length;
        let mut bytes = [0u8; 32];
        bytes[..length].copy_from_slice(&interpreter.contract.input[index..end]);
        B256::new(bytes)
    } else {
        B256::ZERO
    };

    push_b256!(interpreter, load);
}
```

Finally, `COPY` opcodes will take 3 inputs from the stack (memory offset, input data offset, and copy length) and store the corresponding data into memory. Note that it may be necessary to resize memory if the destination of the copied data requires to do so.

```rs
pub fn calldatacopy<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    pop!(interpreter, memory_offset, data_offset, len);
    let len = as_usize_or_fail!(interpreter, len);
    gas_or_fail!(interpreter, gas::verylowcopy_cost(len as u64));
    if len == 0 {
        return;
    }
    let memory_offset = as_usize_or_fail!(interpreter, memory_offset);
    let data_offset = as_usize_saturated!(data_offset);
    memory_resize!(interpreter, memory_offset, len);

    // Can't panic because memory is resized beforehand.
    interpreter.memory.set_data(
        memory_offset,
        data_offset,
        len,
        &interpreter.contract.input,
    );
}
        
pub fn codecopy<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    pop!(interpreter, memory_offset, code_offset, len);
    let len = as_usize_or_fail!(interpreter, len);
    gas_or_fail!(interpreter, gas::verylowcopy_cost(len as u64));
    if len == 0 {
        return;
    }
    let memory_offset = as_usize_or_fail!(interpreter, memory_offset);
    let code_offset = as_usize_saturated!(code_offset);
    memory_resize!(interpreter, memory_offset, len);

    // Can't panic because memory is resized beforehand.
    interpreter.memory.set_data(
        memory_offset,
        code_offset,
        len,
        interpreter.contract.bytecode.original_bytecode_slice(),
    );
}
```


### Stack Instructions

Stack instructions refer to those operations that are solely related to stack manipulation. As such, they only require of interaction with the `Stack` struct of the interpreter.

```rs
// Snippet of the `opcodes!` macro to showcase stack opcodes,
// their mnemonic representation, and their [`Instruction`].
opcodes! {
    0x50 => POP      => stack::pop,
    0x5F => PUSH0  => stack::push0::<H, SPEC>,
    // PUSH (1..32)
    0x60 => PUSH1  => stack::push::<1, H>,
    0x7F => PUSH32 => stack::push::<32, H>,
    // DUP (1..16)
    0x80 => DUP1  => stack::dup::<1, H>,
    0x8F => DUP16 => stack::dup::<16, H>,
    // SWAP (1..16)
    0x90 => SWAP1  => stack::swap::<1, H>,
    0x9F => SWAP16 => stack::swap::<16, H>,
}
```

The `POP` instruction simply calls the `fn pop()` method of the `Stack` struct, and updates the gas costs by calling the `gas!` macro. If there is a stack underflow, it will also update the `InstructionResult`.

```rs
pub fn pop<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    if let Err(result) = interpreter.stack.pop() {
        interpreter.instruction_result = result;
    }
}
```

When the `Stack` struct was initially [introduced in the previous article](../revm-pt1/#stack), only `fn push()` and `fn pop()` were showcased. Nevertheless, another convenient method with the same base principle is available: `fn push_slice()`. This function takes an arbitrary length slice of bytes, and pushes it onto the stack, effectively pushing several words at once. Note that it is necessary to pad the last word (32 bytes) with zeros.

The push operations will push an arbitrary number of words extracted from the contract bytecode into the stack. To do so, they will leverage `fn push_slice()` to seamlessly push several values at once.

```rs
pub fn push<const N: usize, H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    // SAFETY: When analyzing bytecode, trailing bytes are added, so it is safe.
    let ip = interpreter.instruction_pointer;
    if let Err(result) = interpreter
        .stack
        .push_slice(unsafe { core::slice::from_raw_parts(ip, N) })
    {
        interpreter.instruction_result = result;
        return;
    }
    interpreter.instruction_pointer = unsafe { ip.add(N) };
}
```

`PUSH0` is an exception, since it simply pushes a `U256::from(0)` onto the stack without having to read any bytecode.
```rs
pub fn push0<H: Host, SPEC: Spec>(interpreter: &mut Interpreter, _host: &mut H) {
    // Check if the EVM is configured to work with SHANGAI fork
    check!(interpreter, SHANGHAI);
    gas!(interpreter, gas::BASE);
    if let Err(result) = interpreter.stack.push(U256::ZERO) {
        interpreter.instruction_result = result;
    }
}
```

Both `SWAP` and `DUP` opcodes will simply call their respective stack methods. Since these methods weren't explained either, we will do so now.

```rs
impl Stack {
    /// Duplicates the `N`th value from the top of the stack.
    pub fn dup<const N: usize>(&mut self) -> Result<(), InstructionResult> {
        let len = self.data.len();
        if len < N {
            Err(InstructionResult::StackUnderflow)
        } else if len + 1 > STACK_LIMIT {
            Err(InstructionResult::StackOverflow)
        } else {
            // SAFETY: check for out of bounds is done above.
            unsafe {
                let data = self.data.as_mut_ptr();
                core::ptr::copy_nonoverlapping(data.add(len - N), data.add(len), 1);
                self.data.set_len(len + 1);
            }
            Ok(())
        }
    }

    /// Swaps the topmost value with the `N`th value from the top.
    pub fn swap<const N: usize>(&mut self) -> Result<(), InstructionResult> {
        let len = self.data.len();
        if len <= N {
            return Err(InstructionResult::StackUnderflow);
        }
        let last = len - 1;
        self.data.swap(last, last - N);
        Ok(())
    }
}

pub fn dup<const N: usize, H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    if let Err(result) = interpreter.stack.dup::<N>() {
        interpreter.instruction_result = result;
    }
}

pub fn swap<const N: usize, H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    if let Err(result) = interpreter.stack.swap::<N>() {
        interpreter.instruction_result = result;
    }
}
```

### Memory Instructions

Memory instructions refer to those operations that are mainly related to memory manipulation or information.

```rs
// Snippet of the `opcodes!` macro to showcase memory opcodes,
// their mnemonic representation, and their [`Instruction`].
opcodes! {
    0x51 => MLOAD    => memory::mload,
    0x52 => MSTORE   => memory::mstore,
    0x53 => MSTORE8  => memory::mstore8,
    0x59 => MSIZE    => memory::msize,
    0x5E => MCOPY    => memory::mcopy::<H, SPEC>,
}
```

Generally speaking, memory-related instructions are quite straight forward thanks to the manipulation methods offered by the `Memory` struct. Those methods are "wrappers" around the core methods `fn slice()` (getter) and `fn set()` (setter), that handle conversions between types.

```rs
// Load a memory word (32-bytes) at a given index (offset) onto the stack.
pub fn mload<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop!(interpreter, index);
    let index = as_usize_or_fail!(interpreter, index);
    memory_resize!(interpreter, index, 32);
    push!(interpreter, interpreter.memory.get_u256(index));
}

// Stores a stack word (32-bytes) at a given memory index (offset).
pub fn mstore<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop!(interpreter, index, value);
    let index = as_usize_or_fail!(interpreter, index);
    memory_resize!(interpreter, index, 32);
    interpreter.memory.set_u256(index, value);
}

// Stores a stack byte at a given memory index (offset).
pub fn mstore8<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop!(interpreter, index, value);
    let index = as_usize_or_fail!(interpreter, index);
    memory_resize!(interpreter, index, 1);
    interpreter.memory.set_byte(index, value.byte(0))
}

// Push the memory size onto the stack.
pub fn msize<H: Host>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::BASE);
    push!(interpreter, U256::from(interpreter.memory.len()));
}
```

`MCOPY` is a new opcode introduced in the Cancun fork that combines consecutive `MLOAD` and `MSTORE` instructions. The introduction of this new opcode allows for easier and cheaper data copy within memory locations.

```rs
// EIP-5656: MCOPY - Memory copying instruction
pub fn mcopy<H: Host, SPEC: Spec>(interpreter: &mut Interpreter, _host: &mut H) {
    check!(interpreter, CANCUN);
    pop!(interpreter, dst, src, len);

    // into usize or fail
    let len = as_usize_or_fail!(interpreter, len);
    // deduce gas
    gas_or_fail!(interpreter, gas::verylowcopy_cost(len as u64));
    if len == 0 {
        return;
    }

    let dst = as_usize_or_fail!(interpreter, dst);
    let src = as_usize_or_fail!(interpreter, src);
    // resize memory
    memory_resize!(interpreter, max(dst, src), len);
    // copy memory in place
    interpreter.memory.copy(dst, src, len);
}
```

## Moving Forward

And that's it for this deep dive into the EVM interpreter and opcodes. We've covered a lot, but there's always more to explore with Ethereum's execution engine. One major area we haven't touched on yet involves the host-related opcodes, which play a crucial role in how smart contracts interact with the EVM.

Because of the length of the article, I saved that discussion for next time. In our upcoming article, we will cover the host-related opcodes, while diving into contract calls and looking at sub-execution contexts. Our exploration thus far has been guided by certain simplifications and assumptions for the sake of clarity. Yet, as we venture into the intricacies of Ethereum's smart contract interactions and execution models, these foundational concepts will be revisited and expanded.

So, stay tuned! We're just scratching the surface, and there's plenty more to uncover about how one of the most efficient implementations ( _but made in rust ™️_ ) of the EVM works under the hood. See you in the next one!