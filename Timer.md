This article is about a tiny project of using the CMT timer to turn on leds one by one with 1 second interval on the board RSK-RX130.

# Preparation before using CMT timer

## Module Stop State
To reduce power consumption, after resetting, the CMT is in a module stop state.

We need to access the **Module Stop Control Register** to disable/enable the operation of the CMT.
```c
st_system.MSTPCRA.BIT.MSTPA15 = 0;
```

## Register Write Protection
The register write protection feature protects important registers from **being overwritten in case the program malfunctions**.

As we try to write the system register(MSTPCRA), first we need to gain the access to the protected register by rewriting the **PRCR**(Protection Register) register.
```c
SYSTEM.PRCR.WORD = 0xA502;
```
When rewriting the PRCR register, write "A5h" in the upper 8 bits and any value in the lower 8 bits to prevent unintentional writing operations.


