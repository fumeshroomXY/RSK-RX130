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
