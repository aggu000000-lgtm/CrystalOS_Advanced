# Phase 5: Lock-Free Data Structures for IPC

## Overview

This document outlines the design for the Inter-Process Communication (IPC) mechanism between the 7 core agents of CrystalOS. Traditional operating systems rely heavily on locks (mutexes, semaphores) and blocking system calls, which can introduce context-switch overhead and unpredictable latency spikes. To achieve our goal of "Respond in milliseconds" and absolute system fluidly, CrystalOS will abandon these blocking constructs in favor of lock-free Single-Producer Single-Consumer (SPSC) ring buffers.

## Core Architecture: SPSC Ring Buffers

The foundation of our IPC model relies exclusively on SPSC ring buffers established between communicating agents.

### Why SPSC?

The 7-Agent Architecture (Display, Input, Memory/Storage, Network, Audio, Process Control, Security) allows for strict, predetermined communication topologies. Because we know exactly which agents will communicate with one another, we can dedicate pairs of unidirectional SPSC channels instead of complex MPSC (Multi-Producer Single-Consumer) or MPMC structures.

- **Wait-Free Guarantees**: A correctly implemented SPSC ring buffer guarantees wait-free operations for both the producer and the consumer. Neither thread will ever be blocked or suspended waiting for the other.
- **Cache Locality**: SPSC buffers inherently reduce cache coherence traffic. The producer exclusively writes to the `write_index` and the consumer exclusively writes to the `read_index`.
- **Simplicity & Predictability**: SPSC queues have minimal state overhead and execution branches, yielding deterministic millisecond-level execution times.

## Data Structure Design

### The Ring Buffer Structure

```rust
#[repr(C, align(64))] // Align to cache line size to prevent false sharing
pub struct SpscRingBuffer<T, const SIZE: usize> {
    // Written ONLY by the Producer
    write_index: AtomicUsize,

    // Padding to ensure read_index is on a separate cache line
    _pad0: [u8; 64 - core::mem::size_of::<AtomicUsize>()],

    // Written ONLY by the Consumer
    read_index: AtomicUsize,

    // Padding
    _pad1: [u8; 64 - core::mem::size_of::<AtomicUsize>()],

    // The actual data buffer
    buffer: [MaybeUninit<T>; SIZE],
}
```

### Memory Ordering and Synchronization

For the system to remain robust without mutexes, we will strictly rely on atomic memory operations with explicit memory orderings:

1. **Producer Side (Enqueue)**:
   - Read `read_index` with `Acquire` ordering (or cached local copy).
   - Check if the buffer is full (`write_index.wrapping_add(1) % SIZE == read_index`).
   - Write data to `buffer[write_index]`.
   - Update `write_index` with `Release` ordering.

2. **Consumer Side (Dequeue)**:
   - Read `write_index` with `Acquire` ordering.
   - Check if the buffer is empty (`read_index == write_index`).
   - Read data from `buffer[read_index]`.
   - Update `read_index` with `Release` ordering.

### Handling Full Queues

Because we aim for a lock-free design, what happens when a queue is full?

1. **Agent Overload**: If a queue is full, it implies the consumer agent is saturated or hung.
2. **Yield/Backoff**: Instead of blocking, the producer can optionally yield its time slice or spin briefly if in a high-priority context.
3. **Drop Policies (Agent Specific)**: Depending on the agent, certain non-critical packets (e.g., redundant mouse movement updates) can be overwritten or dropped to prioritize the most recent state.

## Integration into the 7-Agent Model

Each agent will have dedicated, memory-mapped shared regions establishing the required pairs of SPSC buffers.

For example, between the **Input Agent** and the **Display Agent**:
- `SpscRingBuffer<InputEvent>` (Input -> Display)
- `SpscRingBuffer<DisplaySync>` (Display -> Input, e.g., for v-blank synchronization to optimize cursor updates).

## Security Boundary

Agent 7 (Security) will validate the creation of these shared memory regions. The SPSC buffers are placed in memory pages that are strictly mapped as:
- **Producer**: Read/Write
- **Consumer**: Read-Only (for the data and `write_index`), Read/Write (only for its own `read_index`).

This ensures that even if an agent is compromised, it cannot arbitrarily manipulate the internal state of another agent via the IPC channel.

## Future Considerations

- **Dynamic Resizing**: For Version 1, ring buffer sizes will be fixed to avoid the complexity and allocation overhead of dynamic resizing. We will profile agent workloads to determine optimal fixed sizes (e.g., power-of-two sizes to allow bitwise masking instead of modulo division).
- **Batching**: Agents can batch reads/writes locally to reduce the overhead of atomic operations when processing high-frequency data streams.
