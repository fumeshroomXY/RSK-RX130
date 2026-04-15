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


# Determine the timer interval
**PCLK** stands for Peripheral Clock.

If PCLK = 32 MHz
- PCLK/8 = 4 MHz -> time tick = 0.25µs
- PCLK/512 = 1/16 MHz -> time tick = 16µs

If expected timer = 1s
- Counter = 1s ÷ 16µs - 1 = 62499


# Timer without interrupt
## Algorithm 1
```c
	int i = 0;
	while(1){
        while(CMT0.CMCNT != CMT0.CMCOR){
        }
        PORT2.PODR.BIT.B1 = 1;
        PORT0.PODR.BIT.B4 = 1;
        PORT0.PODR.BIT.B6 = 1;
        PORT0.PODR.BIT.B7 = 1;
    
        i = (i + 1) % 4;
    
        if(i % 4 == 0){
          //led0
          PORT2.PODR.BIT.B1 = 0;
        }else if(i % 4 == 1){
          //led1
          PORT0.PODR.BIT.B4 = 0;
        }else if(i % 4 == 2){
          //led2
          PORT0.PODR.BIT.B6 = 0;
        }else if(i % 4 == 3){
          //led3
          PORT0.PODR.BIT.B7 = 0;
        }
	}
```
**Problem:** As the CPU clock is much faster than the timer clock, it might read the CMCNT value and execute the led_on part more times before the CMCNT value changes.

This leads to some leds being turned on for a too short time to observe, looking like they are skipped.

## Algorithm 2
```c
	int i = 0, flag = 0;
	while(1){
		//CMT.CMSTR0.BIT.STR0 = 1;
		while(CMT0.CMCNT < CMT0.CMCOR){
			flag = 0;
		}
		//CMT.CMSTR0.BIT.STR0 = 0;
		//CMT0.CMCNT = 0;

		if(flag == 0){
			PORT2.PODR.BIT.B1 = 1;
			PORT0.PODR.BIT.B4 = 1;
			PORT0.PODR.BIT.B6 = 1;
			PORT0.PODR.BIT.B7 = 1;

			i = (i + 1) % 4;
			if(i % 4 == 0){
				//led0
				PORT2.PODR.BIT.B1 = 0;
			}else if(i % 4 == 1){
				//led1
				PORT0.PODR.BIT.B4 = 0;
			}else if(i % 4 == 2){
				//led2
				PORT0.PODR.BIT.B6 = 0;
			}else if(i % 4 == 3){
				//led3
				PORT0.PODR.BIT.B7 = 0;
			}
			flag = 1;
		}
	}
```

Add a flag to ensure that the while loop and the led_on part are executed **alternately**. 

## Algorithm 3
```c
	unsigned short prev = CMT0.CMCNT;
	int i = 0;
	while(1){
		unsigned short now = CMT0.CMCNT;
		if(now < prev){
			PORT2.PODR.BIT.B1 = 1;
			PORT0.PODR.BIT.B4 = 1;
			PORT0.PODR.BIT.B6 = 1;
			PORT0.PODR.BIT.B7 = 1;

			i = (i + 1) % 4;
			if(i % 4 == 0){
				//led0
				PORT2.PODR.BIT.B1 = 0;
			}else if(i % 4 == 1){
				//led1
				PORT0.PODR.BIT.B4 = 0;
			}else if(i % 4 == 2){
				//led2
				PORT0.PODR.BIT.B6 = 0;
			}else if(i % 4 == 3){
				//led3
				PORT0.PODR.BIT.B7 = 0;
			}
		}
		prev = now;
	}
```
Detect counter rollover safely.
# Timer with interrupt
