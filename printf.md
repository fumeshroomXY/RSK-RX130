# Problem of printf()
- Environment: RSK-RX130, e2 studio
- Problem: When I tried to print some debug information to the console, the program behaved weird. 
There was a **huge delay(minutes long)** between when the data changed and when it was updated in printf().
```c
while(1){
  err = R_ADC_Control(0, ADC_CMD_SCAN_NOW, NULL);
  while(R_ADC_Control(0, ADC_CMD_CHECK_SCAN_DONE, NULL) == ADC_ERR_SCAN_NOT_DONE){

  }
  err = R_ADC_Read(0, ADC_REG_CH0, &data);
  printf("value = %d\n", data);
}
```

## printf() is the biggest source of delay
### semihosting

On the RSK_RX130, printf() is typically routed through:
- SCI (UART) at 9600 or 115200 baud, or
- **E1/E2 debugger semihosting**, which is extremely slow

The ADC updates instantly, but you **only see it when printf() finishes**.

It stops the CPU and **waits for the debugger host PC to service the request**.

### buffering
**stdio buffering** makes it worse, prints **pile up** → stall

# printf() inside a callback
Environment: RSK-RX130, e2 studio
```c
volatile uint16_t g_adc_data;

void cb(void *pdata)
{
    printf("value = %d\n", g_adc_data);
}

...
    uint32_t cmt_ch;
    R_CMT_CreatePeriodic(1, &cb, &cmt_ch);

    while (1)
    {
        R_ADC_Control(0, ADC_CMD_SCAN_NOW, NULL);

        while (R_ADC_Control(0, ADC_CMD_CHECK_SCAN_DONE, NULL)
               == ADC_ERR_SCAN_NOT_DONE)
        {
        }
        R_ADC_Read(0, ADC_REG_CH0, (uint16_t *)&g_adc_data);
    }
...
```
Calling printf() from an ISR:
- Is **slow**
- Can **block**, Main loop stops running
- Multiple CMT interrupts **pile up**
- Can cause minutes-long freezes

## Best practice
```c
// In callback:
volatile bool print_flag;
void cb(void *pdata)
{
    print_flag = true;
}

// In main:
if (print_flag)
{
    printf("value = %u\n", g_adc_data);
    print_flag = false;
}
```
- **ISR should not do heavy work** (like printf())


# Semihosting
A debugging technique used in embedded systems that lets code running on a target microcontroller (or processor) use the **host computer’s I/O services**—such as console output, file I/O, or program exit—through the debugger

## How semihosting works
- Your embedded code executes a **special instruction**
- This instruction **halts the CPU**
- The **debugger (GDB, IDE, etc.) intercepts** the request
- The debugger performs the requested operation on the **host OS**
- Control is returned to the target program

## Minimal semihosting example
```c
#include <stdio.h>

int main(void) {
    printf("Hello from semihosting!\n");
    while (1);
}
```
- printf calls a low-level I/O function
- That function executes a semihosting breakpoint
- GDB prints the text in its console

## Advantages
- No extra hardware required
- Extremely easy to use
- Works even when peripherals aren’t configured
- Supports full printf and file I/O

## Disadvantages
- **Very slow** (CPU halts on every call)
- Requires an active debugger
- Not suitable for real-time code
- Code may **crash if semihosting is enabled but no debugger is attached**

## Semihosting vs. UART debugging
|  Feature | Semihosting | UART |
|----------|-------------|------|
|Extra hardware |  No  |Yes    |
|Needs debugger | Yes   |No  |
|Speed | Slow  |Fast|
|Real-time safe |  No  | Yes   |
|Production use |  No   |Yes|
