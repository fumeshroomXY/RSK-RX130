# Pin Direction
- Generally, the pin arrows indicate the input and output directions.
<img src="/micro.png" width="50%">

- Pay attention to the components connected to the pins (**microcontroller or external interface**).
<img src="/applicationheader.png" width="80%">

# DNF(DO NOT FIT)
<img src="/dnf.png" width="80%">


Sometimes a pin(PA0) may connect to two different signal names:
- IO0
- MTIOC4A

Each connection goes through a 0‑ohm resistor:
- R116 = 0Ω
- R10 = 0Ω (DNF) → DNF = Do Not Fit / Not Populated, **not in use by default**

So electrically, **only one path is intended to be connected**, depending on whether the resistor is populated.
- Populate R10 → PA0 connects to IO0
- Populate R116 → PA0 connects to MTIOC4A
- **Never populate both at the same time**(otherwise short two signal nets)

**Why designers do this:**
- Allows board reuse
- Supports late-stage changes
