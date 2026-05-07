# DTC
## SCI without DTC (CPU‑driven transfer)
### How it works
The SCI peripheral raises an **interrupt** (or the CPU polls a status flag) when:
- Data is received (RX)
- The transmit buffer is empty (TX)


The CPU executes an interrupt service routine (ISR) to:
- Read received data from the SCI register
- Write transmit data into the SCI register

### Data flow
```markdown
SCI ⇄ CPU ⇄ RAM
```

### Characteristics
- CPU handles **every byte**
- Simple to configure
- **Higher CPU overhead**, especially at high baud rates
- Interrupt latency can cause data loss if not handled carefully

### Typical use cases

- Low baud rates
- Small data volumes
- Simple applications
- Systems with plenty of CPU headroom

### Practical example
Without DTC
- SCI RX interrupt fires for every byte at 115200 baud
- CPU runs ~115200 interrupts per second
- If another ISR delays execution → RX overrun possible

## SCI with DTC (Data Transfer Controller)
### How it works
- **SCI events (RX/TX) trigger the DTC**
- The DTC **automatically moves data** between SCI data registers and RAM
- The CPU is **not** involved in each byte transfer

### Data flow
```markdown
SCI ⇄ DTC ⇄ RAM
        ↑
       CPU (setup only)
```
The CPU:

- Configures the DTC once (source, destination, transfer size)
- Is interrupted only when:
  - A buffer is full
  - A transfer sequence ends
  - An error occurs
 
### Characteristics

- **Much lower CPU load**
- Predictable timing
- Efficient for continuous or high-speed transfers
- Slightly more complex to configure

### Typical use cases
- High baud rates
- Large or continuous data streams
- Real-time systems
- Low-power designs (CPU can sleep while DTC runs)

### Practical example
With DTC
- SCI RX triggers DTC
- DTC fills a RAM buffer (e.g., 64 bytes)
- CPU interrupted **once per 64 bytes**
- No byte-level timing risk

## Transfer Information Block
```c
dtc_transfer_data_cfg_t td_cfg;
dtc_transfer_data_t transfer_data;
```
`dtc_transfer_data_t` is a structure type defined by the Renesas DTC driver.
- Represents one **DTC Transfer Information block**, which is the data format that the DTC hardware reads.
- Corresponds to the **transfer information memory area** that the DTC fetches when a transfer starts.
- Matches the exact format required by the DTC

`dtc_transfer_data_cfg_t td_cfg` is a software-friendly configuration structure
- Used by the CPU
- Easy-to-understand fields (modes, sizes, enums)
- Developers fill this in by hand

### Typical usage flow (big picture)
``` markdown
td_cfg (human-readable config)
        ↓
R_DTC_Create()
        ↓
transfer_data (hardware format)
        ↓
DTC reads this during transfer
```

### What does `transfer_data` actually contain?
Conceptually, it contains things like:
- Source address
- Destination address
- Transfer count
- Block size
- Chain control bits
- Address mode bits
- Data size bits

These are **encoded into 4 longwords (16 bytes)** in full address mode or 3 longwords (12 bytes) in short address mode.


<img src="/images/AddressMode.png" width="70%">

- Source Address (SAR)
- Destination Address (DAR)

### Why is this variable needed?
The DTC hardware:
- Cannot read your C variables (td_cfg)
- Can only read **raw memory in a fixed format**

So `transfer_data` serves as:
- **The bridge between C code and the DTC hardware engine**


## DTC Vector Table
The table that tells DTC which transfer to execute **when a specific interrupt/event occurs**.

The DTC vector table is **an array in memory** where:

- **Each entry corresponds to one interrupt source**
- Each entry contains the **starting address of a DTC transfer information block**
- When that interrupt occurs, the DTC:
  - Looks up the corresponding vector table entry
  - Fetches the transfer info
  - Automatically transfers data (no CPU ISR needed)
```markdown
Peripheral event (e.g. SCI RX)
        ↓
Interrupt source number
        ↓
DTC vector table lookup
        ↓
Transfer information block
        ↓
Automatic data transfer
```

Think of it as:
- Interrupt vector table → CPU ISR
- DTC vector table → DTC transfer definition

### Relationship to the interrupt vector table
The same interrupt sources are used for CPU interrupts **and** DTC activation.

But when DTC is enabled:
- The interrupt does **not** go to the CPU
- Instead, it triggers a **DTC transfer**

### What is stored in the DTC vector table
Each entry usually contains:
- A **pointer (address)** to a DTC Transfer Information block

|Index |Interrupt source| Value |
|------|  --------------|--------|
|27    | SCI0 RXI       | Address of transfer info  |
|28    | SCI0 TXI       |  Address of transfer info|
|45    | ADC scan end   | Address of transfer info|


<img src="/images/DTCVectorTable.png" width="70%">

## Why SWINT(Software Trigger) is a bad test trigger for DTC
|Trigger | Edge‑triggered | Clearable | Good for DTC testing|
|------|----------------|----------|--------------------|
|SWINT |  ❌ No        |   ❌ No   |    ❌ No  |
|CMT   |  ✅ Yes        |  ✅ Yes   |   ✅ Yes  |
|RTC   |  ✅ Yes       |   ✅ Yes   |   ✅ Yes   |
|SCI RX|  ✅ Yes        |  ✅ Yes    |  ✅ Yes   |

- Software Trigger is one-shot. **No second activation**. No sustained interrupt request.
- Software Trigger cannot be cleared by code and **not retriggerable**.


## Transfer Mode
Example: Copy src -> des[3] (Memory to Memory)
```c
uint32_t src = 1234;
uint32_t des[3];
td_cfg.src_addr_mode = DTC_SRC_ADDR_FIXED;
td_cfg.data_size = DTC_DATA_SIZE_LWORD;
td_cfg.transfer_mode = DTC_TRANSFER_MODE_BLOCK;
td_cfg.dest_addr_mode = DTC_DES_ADDR_INCR;
td_cfg.repeat_block_side = DTC_REPEAT_BLOCK_SOURCE;
td_cfg.response_interrupt = DTC_INTERRUPT_PER_SINGLE_TRANSFER;
td_cfg.chain_transfer_enable = DTC_CHAIN_TRANSFER_DISABLE;
td_cfg.chain_transfer_mode = (dtc_chain_transfer_mode_t)0;

td_cfg.source_addr = (uint32_t)&src;
td_cfg.dest_addr = (uint32_t)des;
td_cfg.transfer_count = 1;
td_cfg.block_size = 1;
```

### `transfer_count`
- One interrupt = one DTC activation
- `transfer_count` = **how many data units are transferred during ONE interrupt**

Example:
```c
transfer_count = 8;
mode = Normal;
```

```markdown
Interrupt #1:
  DTC transfers 8 elements (buffer[0]..buffer[7])
  transfer_count reaches 0
  DTC disables itself

Interrupt #2:
  ❌ DTC does nothing
```
- Only one interrupt was needed
- All 8 transfers happened **within that single interrupt**

### `repeat_block_side`
In Repeat mode, only ONE side resets:
- Source repeat **or**
- Destination repeat

If you accidentally set:
- Source = repeat
- Destination = increment only

Then the destination **walks through memory**(silent memory corruption).

Correct setting for register → array:
```markdown
Source: FIXED
Destination: INCREMENT + REPEAT
```

### `response_interrupt`
#### Trigger interrupt
- Comes from a peripheral (e.g., CMT0, ADC, SCI RXI)
- Starts (activates) the DTC transfer

#### DTC interrupt (different from Trigger interrupt)
- Comes from the DTC module
- Happens after data transfer
- Optional
- Used only if you want CPU notification
```c
DTC_INTERRUPT_AFTER_ALL_COMPLETE;   // One CPU interrupt is generated after the entire transfer finishes.
DTC_INTERRUPT_PER_SINGLE_TRANSFER;  // The DTC generates a CPU interrupt after each single transfer.
```

Notice:
- Sometimes the DTC interrupt vector is mapped to **the SAME ISR** as the trigger interrupt or the DTC interrupt itself is considered the same as the trigger interrupt by MCU.

This will cause:
```markdown
Trigger interrupt occurs (1 time)
        ↓
DTC activated
        ↓
Transfer #1 → DTC interrupt -> Trigger interrupt ISR
Transfer #2 → DTC interrupt -> Trigger interrupt ISR
Transfer #3 → DTC interrupt -> Trigger interrupt ISR


Trigger interrupt ISR should be called once
                ↓
Trigger interrupt ISR is called more than once
```


### Normal Mode
- Performs a **one‑shot transfer per launch**.
- Each DTC activation transfers all configured transfer data once.
- After completing the transfer, the DTC **stops and disables** itself for that vector unless re-enabled.
- Further interrupts do **NOT** trigger DTC. **Only the CPU ISR runs** (if enabled)

```markdown
transfer_count = 1;
1 trigger interrupt, 1 DTC interrupt:
src -> des[0]
des[1], des[2] do not change.

transfer_count = 3;
DTC_INTERRUPT_PER_SINGLE_TRANSFER;
1 trigger interrupt, 3 DTC interrupts:
interrupt 0: src -> des[0]
interrupt 1: src -> des[1]
interrupt 2: src -> des[2]

transfer_count = 3;
DTC_INTERRUPT_AFTER_ALL_COMPLETE;
1 trigger interrupt, 1 DTC interrupt:
src -> des[0]
src -> des[1]
src -> des[2]
```

### Repeat Mode
- **Automatically** repeats the same transfer
- After completing a block transfer, the transfer count is **reloaded**.
- One address (source or destination) is **fixed**, the other **increments normally and should be repeated**.
```markdown
transfer_count = 1;
DTC_INTERRUPT_PER_SINGLE_TRANSFER;
DTC_REPEAT_BLOCK_SOURCE;
Every trigger interrupt:
interrupt 0: src -> des[0]
interrupt 1: src -> des[1]
interrupt 2: src -> des[2]
...
(silent memory corruption)

transfer_count = 1;
DTC_INTERRUPT_PER_SINGLE_TRANSFER;
DTC_REPEAT_BLOCK_DESTINATION;
Every trigger interrupt:
src -> des[0]
des[1], des[2] do not change.

transfer_count = 3;
DTC_INTERRUPT_PER_SINGLE_TRANSFER;
DTC_REPEAT_BLOCK_DESTINATION;
Every three trigger interrupts:
trigger interrupt 0: src -> des[0]
trigger interrupt 1: src -> des[1]
trigger interrupt 2: src -> des[2]

transfer_count = 3;
DTC_INTERRUPT_AFTER_ALL_COMPLETE;
DTC_REPEAT_BLOCK_DESTINATION;
Every trigger interrupt:
src -> des[0]
src -> des[1]
src -> des[2]
```

### Block Mode
Transfers one block of data per launch.
```markdown
if transfer_count = 1, block_size = 3:
Every trigger interrupt:
src -> des[0]
src -> des[1]
src -> des[2]

if transfer_count = 1, block_size = 1:
Every trigger interrupt:
src -> des[0]
des[1], des[2] do not change.
```

## The Order of DTC Transfer and ISR
Example: If a CMT0 interrupts occurs, which one goes first? DTC transfer or ISR?
Answer: **The DTC transfer runs first**. The CMT0 ISR executes afterward (if it is enabled).

### DTC transfer (if enabled for that interrupt)
- The ICU sees that DTC is linked to the CMT0 interrupt source
- The DTC is invoked immediately
- The DTC performs all configured transfers
- This happens without entering an ISR
- The CPU is **not involved**

### CPU interrupt (ISR), if enabled
- After the DTC finishes, the CPU:
  - Stacks registers
  - Jumps to the CMT0 ISR
- If the CMT0 interrupt is masked (IEN = 0), this step is skipped

### Why this ordering exists
On RX MCUs: DTC is treated as a **hardware “pre‑ISR” mechanism**.

The DTC is designed to:

- Move data **before** the CPU touches anything
- **Reduce ISR execution time**
- Let ISRs operate on **already‑updated data**

This is why DTC is commonly used with:
- ADC end interrupts
- SCI RX interrupts
- Timers (CMT)
- RTC ticks

