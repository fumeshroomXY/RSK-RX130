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




# ADC Input on the RSK‑RX130 Board without external hardware
The board includes a **variable resistor (VR)** connected to an ADC pin(AN000).

So we can:
- Rotate the knob
- Read voltage changes via ADC
