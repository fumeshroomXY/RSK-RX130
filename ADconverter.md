# A/D converter
## Key characteristics

### Resolution
- Measured in bits (e.g., 8‑bit, 10‑bit, 12‑bit)
- Higher resolution = more precise measurement
- Example:
  - 8‑bit ADC → 256 possible values
  - 12‑bit ADC → 4096 possible values

### Sampling rate
- How often the signal is measured (e.g., samples per second)
- Higher rate = better representation of fast‑changing signals

## Channel
ADC channel = **one specific analog input path (pin) connected to the ADC**

You choose which room to measure by selecting a channel.

On the Renesas RX130, ADC channels are named like: AN000, AN001, AN003, etc.

Each one is:
- A separate analog pin
- A different voltage source


Even though there are many channels, there is usually **only ONE ADC unit inside**.

### Single channel vs multiple channels
Single channel
- Read one analog signal repeatedly
- Example: potentiometer or sensor

Multiple channels
- Read several signals **one after another**, the ADC switches between them very fast.
- Example:
  - AN000 → temperature
  - AN001 → voltage
  - AN002 → current

## Scan Mode
Scan mode means that the converter **automatically converts one or more selected channels in sequence**.
### Single Scan Mode
- ADC converts the selected channel(s) once
- Then stops
- Example:
  - Convert AN000 one time
  - Or convert AN000 → AN001 → AN002 one time

### Continuous Scan Mode
- ADC keeps scanning **repeatedly**
- Stops only when you tell it to stop

```markdown
AN000 → AN001 → AN002 → AN000 → AN001 → ...
```

### Why scan mode is useful
- Less software code
- Consistent timing between channels
- Better for real-time systems
- Easier to synchronize with timers or interrupts

## Internal Reference Voltage
The internal reference voltage (**Vref**) is a precise, built‑in voltage source inside the microcontroller that the ADC uses as **a comparison standard** when converting an analog signal into a digital value.

An ADC does **relative measurement**, not absolute measurement.

For an N‑bit ADC:
ADC value = **Vin / Vref ×(2^N−1)**

### Internal vs External reference voltage
Internal reference voltage
- Generated inside the MCU
- Fixed value (for example: 1.4 V, 2.5 V, etc., depending on MCU)
- No external components needed

External reference voltage
- Provided by an external pin
- Higher accuracy, more cost and complexity

### How the reference voltage affects the ADC
- Defines the measurement range
- Affects resolution (volts per step), **Lower reference voltage → finer resolution**

### Commone reference voltage
- AVCC (supply voltage, e.g. 3.3V)
  - You usually know it from the board schematic
- Internal reference voltage(most common for sensors)
  - Vref is a fixed, known value
  - Specified in the MCU datasheet
- External precision reference (best accuracy)

### How the voltage is converted into the temperature
Example: Temperature sensor (very typical)

Assume a temperature sensor with this specification:
```markdown
Output = 10 mV per °C
0 °C = 0 V
```
If ADC reads 682:

Vinput = 682 / 4095 × 1.5 ≈ 0.25 V

Temperature = Vinput​ / 10 mV = 25 °C​

## Alignment
When an ADC finishes a conversion, it produces an N‑bit number (for example, 12 bits).

Alignment controls **where those N bits are placed in the register**.

Example 1: 12‑bit ADC, left‑aligned in 16 bits
```markdown
Register bits:  [15 14 13 12 11 10  9  8 | 7  6  5  4  3  2  1  0]
ADC value:      [ D11 D10 D9  D8  D7 D6 D5 D4 D3 D2 D1 D0 | 0  0  0  0 ]
```
- ADC data starts at bit 15
- Lower bits are filled with zeros
- Numeric value is shifted left


Example 2: 12‑bit ADC, right‑aligned
```markdown
Register bits:  [15 14 13 12 | 11 10  9  8  7  6  5  4  3  2  1  0]
ADC value:      [ 0  0  0  0 | D11 D10 D9 D8 D7 D6 D5 D4 D3 D2 D1 D0 ]
```

### Why left‑alignment exists
1. Easy 8‑bit usage
If you only need **8‑bit precision**, left alignment lets you just read the **high byte**:
```c
uint8_t value8 = (ADCR >> 8);
```
No extra shifting needed.

## Trigger
A trigger is the event or signal that tells the ADC **when to sample the analog signal**.

### Types of ADC triggers
#### Software trigger
You start conversion using code.

Example:
```c
S12AD.ADCSR.BIT.ADST = 1;   // Start ADC conversion
```

#### Hardware trigger
An external or internal hardware event starts the ADC.
Common trigger sources:
- Timer overflow
- Compare match event
- Interrupt event(Timer)

Advantages:
- Precise timing
- Low CPU overhead
- Ideal for real‑time systems

# ADC Input on the RSK‑RX130 Board without external hardware
The board includes a **variable resistor (VR)** connected to an ADC pin(AN000).
<img src="/images/AN000.png" width="50%">

So we can:
- Rotate the knob
- Read voltage changes via ADC
