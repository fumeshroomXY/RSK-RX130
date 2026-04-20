# Single step VS. Run freely
## Single step
When you run a program with breakpoints or single‑step:
- The CPU is **forced to stop frequently**
- The debugger must:
  - **Save the full CPU context**
  - **Spill registers to memory**
- **Interrupts** occur at completely different relative times
- printf() **semihosting** suddenly “works”

So the race condition disappears temporarily and the program behaves as if everything were correct.

This is a textbook **Heisenbug: observing it changes its behavior**.

This makes stepping **especially misleading**.

## Why this is especially visible in embedded systems
- **No operating system** to isolate effects
  - No scheduler
  - No virtual memory
  - No thread abstraction
- Interrupts make timing critical
  - In embedded systems: Main code, ISRs, DMA and Hardware state machines, all interact concurrently.
  - Single stepping can fix race conditions.
- Semihosting and I/O amplify the effect
