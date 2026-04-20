# Warning: Function "R_CMT_CreatePeriodic" declared implicitly
This is a classic C compiler error, especially in embedded projects.

The compiler is telling you:

“You are calling a function named R_CMT_CreatePeriodic(), but I have **never seen a declaration** for this function before.”

So the compiler:
- Does not know the function’s prototype
- Does not know its parameters
- Does not know its return type

**Fix:** Include the correct header file
```c
#include "r_cmt_rx_if.h"
```

# Warning: Argument of type "volatile uint16_t *" is incompatible with parameter of type "uint16_t *const"
This happens when you call something like:
```c
volatile uint16_t adc_data;
R_ADC_Read(0, ADC_REG_CH0, &adc_data);
```
This means that the value this pointer points to may change at any time.

And the function prototype is:
```c
adc_err_t R_ADC_Read(uint8_t unit,
                     adc_reg_t reg,
                     uint16_t * const p_data);
```
It means:
- uint16_t * → pointer to **non‑volatile** uint16_t
- const applies to the **pointer**, not the data
- The function promises not to reassign p_data, but may read/write *p_data

C **does not allow dropping volatile qualifiers** implicitly.

You are attempting: volatile uint16_t *  →  uint16_t *

That is unsafe, so the compiler complains.

## Why write the API this way
The ADC driver authors assumed:
- The ADC writes to **ordinary RAM**
- The buffer itself does **not** need to be volatile
- Only **registers** are volatile, not result buffers

## Correct, clean solutions
### Solution 1 (BEST practice): use a non‑volatile buffer
```c
uint16_t adc_data;   // NOT volatile
R_ADC_Read(0, ADC_REG_CH0, &adc_data);
```

### Solution 3: cast explicitly (allowed, but be careful)
```c
R_ADC_Read(0, ADC_REG_CH0, (uint16_t *)&adc_data);
```
- You are promising the compiler this is safe
- Use only if you understand the implications
