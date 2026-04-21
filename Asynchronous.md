# Asynchronous serial communication
A mechanism that allows the transmitter and receiver to stay synchronized **without a shared clock**.

In asynchronous serial communication:
- **No common clock line** between sender and receiver
- Only TX, RX, (and usually GND) are shared
- Timing is recovered from the data stream itself

## Frame
Each transmitted character is framed like this:
```markdown
Idle | Start | Data bits | Optional parity | Stop | Idle
```
- Idle line = logic **HIGH (1)**
- Start bit = logic LOW (0)
  - **High -> Low, falling edge** → start bit
  - Allows the receiver to samples data bits at the center of each bit
- Data bits
  - Typically 7 or 8 bits
  - Sent **LSB first**
- Stop bit(s) = logic HIGH (1)
  - Ensures line returns to **idle state**
  - If the stop bit is not detected correctly → **framing error**
  - The receiver uses **timing(fixed intervals), not voltage meaning** to recognize the stop bit

## Baud rate
Baud rate is the number of signal changes (symbols) transmitted per second.

In most embedded serial systems:
- Baud rate = bits per second (bps)
- This is true because **each symbol represents exactly one bit** in UART.


In Other Systems:
- one symbol may represent multiple bits
- Baud rate ≠ Bit rate

### Why Both Devices Must Use the Same Baud Rate
If transmitter and receiver baud rates differ:
- Sampling occurs too early or too late
- Bits shift positions
- Stop bit may be misread
- **Framing and parity errors occur**

### How Baud Rate Is Generated in Microcontrollers
```markdown
System clock → Prescaler → Baud rate generator
```
Baud rate = fCPU​ / (N × Divider)
​- fCPU = clock frequency
- N = oversampling factor (8 or 16)

## Parity
A simple **error‑detection mechanism** added to each data frame to help detect single‑bit transmission errors.

Parity is **an extra bit** appended to the transmitted data that indicates whether the number of 1 bits in the data is **even or odd**.

The receiver recalculates parity and compares it with the received parity bit to check if an error likely occurred.

### Types of Parity
#### Even Parity
The total number of 1 bits including the parity bit is even.
```markdown
Data = 1011001  (four 1s → even)
Parity bit = 0 (keeps total even)
```
If data has an odd number of 1s, parity bit = 1.
#### No Parity (Most Common)
- No parity bit is sent
- Higher data rate
- Less error checking

# Asynchronous vs Synchronous Summary
| Feature  | Asynchronous (UART) | Synchronous (SPI, I²C)|
|------|-----|-----|
|Clock line | ❌ No  |✅ Yes  |
|Start bit  | ✅ Yes | ❌ No |
|Stop bit  | ✅ Yes  |  ❌ No |
|Synchronization  | Per frame | Continuous | 
|Complexity  |  Simple | Faster, more wires|
