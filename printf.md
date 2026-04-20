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
1. semihosting

On the RSK_RX130, printf() is typically routed through:
- SCI (UART) at 9600 or 115200 baud, or
- **E1/E2 debugger semihosting**, which is extremely slow

The ADC updates instantly, but you **only see it when printf() finishes**.

It stops the CPU and **waits for the debugger host PC to service the request**.

2. **stdio buffering** makes it worse, prints **pile up** → stall

