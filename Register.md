# How to R/W registers
```c
struct st_port2 {
	union {
		unsigned char BYTE;
		struct {
			unsigned char B7:1;
			unsigned char B6:1;
			unsigned char B5:1;
			unsigned char B4:1;
			unsigned char B3:1;
			unsigned char B2:1;
			unsigned char B1:1;
			unsigned char B0:1;
		} BIT;
	} PDR;
  ...
}
```
For a register like above, there are two ways to set:
- set the bit directly
```c
st_port2.PDR.BIT.B0 = 1U;
```

- set the whole register by using the unsigned
```c
st_port2.PDR.BYTE = 0x01;
// same as st_port2.PDR.BIT.B0 = 1U;
```

You **CANNOT** set the *BIT* like this:
```c
st_port2.PDR.BIT = 0x01;
```
Because *PDR.BIT* is a struct, but the *=* operator requires an **integer**. 

# Input/Output Register
- Input Register

An input register is used to receive and temporarily store data **coming into** a system or processing unit.

- Output Register

An output register is used to store data that is being **sent out** of the system or processing unit.

## I/O Port Registers
Some registers can be defined as input or output because they are bidirectional storage elements whose role is **determined by control signals and system context.**
- Each pin has a **data register**
- A **direction register** defines behavior

```c
struct st_port2 {
	union {
		unsigned char BYTE;
		struct {
			unsigned char B7:1;
			unsigned char B6:1;
			unsigned char B5:1;
			unsigned char B4:1;
			unsigned char B3:1;
			unsigned char B2:1;
			unsigned char B1:1;
			unsigned char B0:1;
		} BIT;
	} PDR;  // direction register
	char           wk0[31];
	union {
		unsigned char BYTE;
		struct {
			unsigned char B7:1;
			unsigned char B6:1;
			unsigned char B5:1;
			unsigned char B4:1;
			unsigned char B3:1;
			unsigned char B2:1;
			unsigned char B1:1;
			unsigned char B0:1;
		} BIT;
	} PODR;  // data register
	char           wk1[31];
	union {
		unsigned char BYTE;
		struct {
			unsigned char B7:1;
			unsigned char B6:1;
			unsigned char B5:1;
			unsigned char B4:1;
			unsigned char B3:1;
			unsigned char B2:1;
			unsigned char B1:1;
			unsigned char B0:1;
		} BIT;
	} PIDR;  // data register
...
}
```

```c
PDR bit = 0 → Input
PDR bit = 1 → Output
```
If PDR is not set, the meaning of PODR and PIDR is not defined.

```c
PDR = 1, PODR = 1 → High output from the pin
PDR = 1, PODR = 0 → Low output from the pin
```
If PDR is 0 (input), writing to PODR does **not** change the pin voltage


```c
PDR = 0
PIDR == 1 → Pin is High
PIDR == 0 → Pin is Low
```

## When the LED is wired active‑low
```markdown
VCC ── resistor ── LED ── GPIO
```
In this case:
- GPIO = 0 → LED ON
- GPIO = 1 → LED OFF
