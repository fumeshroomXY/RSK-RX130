# callback function
The system (or driver) will call the function **automatically** later when a specific event occurs.

```c
bool R_CMT_CreatePeriodic(
    uint32_t frequency_hz,
    void (*callback)(void *pdata),
    uint32_t *channel
);
```

```c
void (*callback)(void *pdata)
```
This is a function pointer. 
It means:

“When the timer fires (at the given frequency), **call this function.**”

The callback must match this signature:
```c
void my_callback(void *pdata)
```
- void → returns nothing
- void *pdata → generic pointer for optional user data, lets you pass any data you want to the callback.
```c
// × THIS IS WRONG
R_CMT_CreatePeriodic(1000, timer_callback(&my_data), &channel);  
// Because that would call the function immediately and pass its return value (which is void — invalid).

// Correct:
volatile uint32_t ms_ticks = 0;

void systick_callback(void *pdata)
{
    uint32_t *tick = (uint32_t *)pdata;
    (*tick)++;
}

/* Setup */
R_CMT_CreatePeriodic(1000, systick_callback, &channel);
/* pdata internally points to &ms_ticks */
```

