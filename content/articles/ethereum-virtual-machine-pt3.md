---
title: "Ethereum Virtual Machine (in Rust) - Part 3"
date: 2024-03-16T16:00:00+00:00
slug: revm-pt3
category: article 
summary: Understanding calls and sub-execution contexts.
description: "In this second article of the revm series, we will dive deeper into the EVM interpreter. We will understand how it works, especially focusing on the different opcodes."
cover:
  image:
  alt:
  caption: 
  relative:
showtoc: true
draft: false
---

The [previous article](../revm-pt2) focused on the structure and design of the EVM interpreter, as well as its opcodes. By now, you should have a solid understanding of its implementation and mechanics. Nevertheless, due to bigger complexity and the size constraints for a single article, two key opcode groups were not discussed: control flow, and host-related opcodes.

In this article we look into these opcode groups, and we will also enhance the current interpreter —and its components—, in order to implement the missing EVM features.

# Sub-execution contexts

Despite mentioning contract calls, so far our discussion on EVM execution has operated under the implicit assumption of a singular execution context. This simplification is great at helping absorb the core concepts of the execution model, but falls short to explain the contract interactions that are fundamental to Ethereum's functionality. Allowing calls among contracts requires awareness of nested calls, isolated memory management, and result handling. To incorporate this requirements, we must introduce the concept of sub-execution contexts, or "frames" as defined in the `revm` repo.

Introducing frames marks a paradigm shift in how we perceive EVM execution. Think of frames as layers of abstraction, each encapsulating a distinct execution context. This new model doesn't replace our existing interpreter; instead, it adds depth, accommodating the nested nature of contract calls. Imagine peeling back layers of an onion, where each layer represents a frame—a self-contained execution environment, complete with its own interpreter instance.

This conceptual shift necessitates a dual-loop execution model within the EVM:

The first loop is call loop that everything starts with, it creates call frames, handles subcalls, it returns outputs and calls Interpreter loop to execute bytecode instructions. It is handled by ExecutionHandler.

- The call loop, a top-level loop manages the overall transaction. It creates (call)frames and iterates through each of them, handling subcalls and its outputs.
- Within each individual frame, a dedicated interpreter loop handles the bytecode execution specific to that context. This is the execution loop that we are already familiar with.

Additionally, despite each frame runs its own interpreter, a common host and a shared memory accross all instances are employed. As such, the `Memory` struct that we devined in the [first article](../revm-pt1/#memory) must be reimplemented.

{{< figure src="/blog/images/revm/frames-diagram.png" align=center caption="_EVM execution model showcasing how the different components interact with each other._" >}}

The following sections will introduce new structs —or enhanced previously defined ones—, to set the missing pieces of the EVM execution workflow.

## Shared Memory

In order to enhance the previous memory definition, two new fields will be added to the struct: `checkpoints` and `last_checkpoint`. Combined, these two fields allow for a shared usage of the `buffer` among all interpreter instances.

As a rule of thumb, all functions of the `SharedMemory` implementation will remain the same as those defined in `Memory`. The following snippets covers those that were previously simplified.

```rs
pub struct SharedMemoryMemory {
    /// The underlying buffer.
    buffer: Vec<u8>,
    /// Memory checkpoints for each depth.
    checkpoints: Vec<usize>,
    /// Invariant: equals `self.checkpoints.last()`
    last_checkpoint: usize,
    /// Memory limit. See [`CfgEnv`](revm_primitives::CfgEnv).
    #[cfg(feature = "memory_limit")]
    memory_limit: u64,
}

impl SharedMemory {
    /// Creates a new memory instance with the given `capacity`.v
    pub fn with_capacity(capacity: usize) -> Self {
        Self {
            buffer: Vec::with_capacity(capacity),
            checkpoints: Vec::with_capacity(32),
            last_checkpoint: 0,
            #[cfg(feature = "memory_limit")]
            memory_limit: u64::MAX,
        }
    }
}
```

Since data from several execution contexts colives within the same buffer, auxiliary functions to manage the checkpoints are required for proper memory management.

```rs
impl SharedMemory {
    /// Prepares the shared memory for a new context.
    pub fn new_context(&mut self) {
        let new_checkpoint = self.buffer.len();
        self.checkpoints.push(new_checkpoint);
        self.last_checkpoint = new_checkpoint;
    }

    /// Prepares the shared memory for returning to the previous context.
    pub fn free_context(&mut self) {
        if let Some(old_checkpoint) = self.checkpoints.pop() {
            let last = self.checkpoints.last().cloned().unwrap_or_default();
            self.last_checkpoint = last;
            // SAFETY: buffer length is less than or equal `old_checkpoint`
            unsafe { self.buffer.set_len(old_checkpoint) };
        }
    }

    /// Returns the length of the current memory range.
    pub fn len(&self) -> usize {
        self.buffer.len() - self.last_checkpoint
    }

    /// Resizes the memory in-place so that `len` is equal to `new_len`.
    pub fn resize(&mut self, new_size: usize) {
        self.buffer.resize(self.last_checkpoint + new_size, 0);
    }
}
```

As previously explained, since memory is a word-addressable byte array, its getter and setter methods require an `offset` and a `size`. Additionally, the shared natured of the struct, also requires the usage of the `last_checkpoint` to properly manage the data buffer.

```rs
impl Memory {
    /// Returns a byte slice of the memory region at the given offset.
    /// Panics on out of bounds.
    pub fn slice(&self, offset: usize, size: usize) -> &[u8] {
        let end = offset + size;
        let last_checkpoint = self.last_checkpoint;

        self.buffer
            .get(last_checkpoint + offset..last_checkpoint + offset + size)
            .unwrap_or_else(|| {
                debug_unreachable!("OOB: {offset}..{end}; len: {}", self.len())
            })
    }
}
```

## InterpreterAction and InterpreterResult

```rs
pub struct InterpreterResult {
    /// The result of the instruction execution.
    pub result: InstructionResult,
    /// The output of the instruction execution.
    pub output: Bytes,
    /// The gas usage information.
    pub gas: Gas,
}

pub enum InterpreterAction {
    /// CALL, CALLCODE, DELEGATECALL or STATICCALL instruction called.
    Call {
        /// Call inputs
        inputs: Box<CallInputs>,
        /// The offset into `self.memory` of the return data.
        /// This value must be ignored if `self.return_len` is 0.
        return_memory_offset: Range<usize>,
    },
    /// CREATE or CREATE2 instruction called.
    Create { inputs: Box<CreateInputs> },
    /// Interpreter finished execution.
    Return { result: InterpreterResult },
    /// No action
    #[default]
    None,
}
```