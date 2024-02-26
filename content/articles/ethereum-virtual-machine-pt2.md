---
title: "Ethereum Virtual Machine (in Rust) - Part 2"
date: 2024-02-21T20:00:00+00:00
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

```md
Interpreter
  - [X] Contract / Bytecode states
    - [X] Jump Map
  - Opcodes
    - [X] opcode::{Instruction, OpCode, OPCODE_JUMPMAP};
    - implement opcodes
        - instruction
        - macros
  - Call Opcodes
  - Gas
  - Traces?
```

# Interpreter

As uncovered in the [previous article](../revm-pt1) —which I highly recommend checking out if you haven't yet-, the interpreter is the core engine of the EVM. It is in charge of running the execution loop that processes and executes each of the instructions stored in the bytecode. Additionally, besides the computation and returning the results of the execution, the interpreter also needs to keep track of the gas consumption.

To fullfill all of its tasks, the interpreter will use its attributes (contract bytecode, stack, memory, etc.) and also interact with the EVM host.

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
Before digging into `BytecodeLocked`, let's look at a couple of initialization functions for the `Contract` struct; A part from the standard implementation of the `fn new()` method, it also implements a convenient function `fn new_env()` to create a new contract instance from the current EVM environment.

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
 Additionally, analysed bytecode can be converted into `BytecodeLocked`, signaling it has passed all checks and analyses, locking it against further modifications. Then, it will be feedable to the interpreter's `Contract`.

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

## Opcodes and Instructions

As you may have already anticipated, efficient methods for associating opcodes with their corresponding execution logic and mnemonic representation are essential. `revm` achieves it thanks to the creation of an `InstructionTable` and an `OPCODE_JUMPMAP`.

### Basics Types

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

`revm` also implements a Boxed version of both types. By doing so, devs who want to implement their own even, can have closures available for `BoxedInstruction` at the expense of some performance. Because of that, both implementations are wrapped under an enum.

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

### More types: Opcode Information

### Logic Implementation

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

#### Arithmetic
---
---