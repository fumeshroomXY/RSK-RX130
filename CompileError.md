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
