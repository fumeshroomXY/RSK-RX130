# How to determine the clock cycle
For example, if we want to use the CMT timer, since its clock cycle is determined by **PCKB**, we need to figure out PCLKB.

<img src="/clock.png" width="80%">

- In the above picture, we need to calculate what is the final clock cycle for PCKB by figuring out the output of corresponding **selecters, frequency dividers and the original clock.**
- When the board is reset, we need to pay attention to the **original value** of the related registers.


<img src="/SCKCR3.png" width="80%">
- For example, for the SCKCR3 register, CKSEL == 0x000 means that LOCO clock is selected.

- Because the reset value may be different from what is written in the user manual, it's better to **check the real value when debugging**.

## Smart Configure Clock
We can also use **Smart Configurator** in e2 studio to configure the clock conveniently.


<img src="/smartconfigureclock.png" width="80%">
