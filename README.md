# STM32 debugging with SWV trace

*There is a simple and elegant way to debug an STM32 system without using any UART. It only requires an extra ST-Link pin.*

## The problem

Standard output peripherals, like UART, may be used to help with debugging MCUs. However, this solution can became intrusive for the following reasons:
- in interrupt mode, the frequency of interrupts may be very high, resulting in performance drop
- in polling mode, the waiting time for transmission affects the performance even more

## How it works

The SWO (Serial Wire Output) trace pin is a hardware feature of the ARM Cortex-M3/M4/M7 processors that addresses this problem. It allows tracing of system activity and memory without significant impact on performance.

It uses the ITM (Instrumentation Trace Macrocell) module of the Cortex CoreSight technology, a part of the Cortex-M debugging infrastructure. It’s like a serial port built into the CPU that allows you to send data (for example, strings or variables) to a debugging tool via the SWO pin. You don't need any UART or extra pins. Only a standard SWD connection plus the SWO pin and some software settings are required.

The ITM has 32 communication channels. ITM Channel 0 is used for printf-style output via the debug interface.

> [!WARNING]
> Cortex-M0 and Cortex-M0+ based MCUs do not have SWO capabilities, nor do the boards on their base, such as Nucleo 32.

## Hardware list

- **STM32 development board** from the Black Pill or Nucleo series can be used. The [WeActStudio BluePill Plus](https://github.com/WeActStudio/BluePill-Plus) board is a good choice. It's an improved version of the popular Blue Pill board, featuring a genuine STM32F103C8T6 chip.
  
- **ST-Link debugger** / programmer board, original or compatible. [WeActStudio MiniDebugger](https://github.com/WeActStudio/WeActStudio.MiniDebugger) is recommended for use with the BluePill Plus board. It is compatible with the original ST-Link and has both SWD and UART interfaces.

- **Jumper wires** - 5 pieces in total may be needed: 4 for the SWD connector and 1 for the additional SWO pin. The amount and type of wires used depend on the board you have.  For the BluePill Plus board, installed on the breadboard and connected to the MiniDebugger (as shown in the picture), you need one 10-cm-long male-to-male jumper wire for the SWO connection.  Four SWD jumper wires are already combined with the debugger's connector and can be directly connected to the board's SWD connector. For the Nucleo board, no wires are needed, as all the connections are made inside the board. However, not all Nucleo boards support SWO.

- **USB Type-C cable** is needed to connect the debugger to a PC. Choose the cable's length according to your needs.

> [!WARNING]
> Adding trace support to ST-Link clone dongles requires soldering a wire to the chip inside the dongle. See the references for details.

<!-- <p align="center">
<img src="images/board-debugger-connection.jpg" width="600px">
</p> -->

## How to connect

- Connect SWD connector of the STM32 board (4 pins) to the corresponding wires of the debugger (4 jumper wires).
- Connect SWO pin of the board (pin PB3 of the STM32F103C8T6 chip and pin B3 on the BluePill Plus board) to the SWO pin (jumper wire) of the debugger.

# Software setup

**STM32CubeIDE** has to be installed.

1. In the STM32CubeIDE Device Configuration Tool in Pinout & Configuration click on System Core -> SYS and set the Debug method to Trace Asynchronous Sw. This enables the SYS_JTDO-TRACESWO pin on your microcontroller (pin PB3 of the STM32F103C8T6 chip).

2. Enable SWV in STM32CubeIDE. Open the Debug Configurations... under the Debug icon (looking as a little beetle). In the Debugger tab, enable Serial Wire Viewer (SWV) and make sure the Core Clock is set to your system clock (which you can find in Device Configuration Tool -> Clock Configuration -> SYSCLK).

3. Click on Debug icon, switch to the Debug perspective, click on Window -> Show View -> SWV -> SWV ITM Data Console to be able to view the SWV output.

4. While the debugger is paused on the first breakpoint, bring up the SWV settings by clicking on the wrench icon in the upper-right corner of the SWV ITM Data Console tab. In the bottom part of the window, under ITM Stimulus Ports, check pin 0. 

5. Press the red circle button in the SWV ITM Data Console and then press the Resume button on the Toolbar to run the code and start tracing.

6. You should see messages printed in the console.

7. You can also monitor the value of the global variable (`count` in the used code)  by clicking on 'Window -> Show View -> Live Expressions, checking the tab and entering the variable name in the 'Expression' column.

8. Press the Suspend button on the Toolbar to pause the program and the Terminate button to exit the Debug perspective. 


## What the code does

- **bluepill-plus_swv_putchar**
The simplest way to print something to the console seems to be using the ubiquitous `printf` function. But in the case of the code running on a microcontroller, it's won't be that easy. We should redirect `printf` to SWO first.  
To do this, we'll be using `ITM_SendChar` function, which transmits a character via the ITM channel 0. This function is part of the `__io_putchar` function, which is included in `_write` function in the `syscalls.c` file.  
After including `<stdio.h>` and the `__io_putchar` to the code, we can start using `printf`, as we usually do in any C program. The code here outputs the counter's values to the console once per second in an endless loop.

- **bluepill-plus_swv_write**
If the `syscalls.c` file is missing or not implemented, the entire `_write` function must be overwritten. In this case `ITM_Send Char` is inside this `_write` function, which we insert into the code instead of `__io_putchar`. The other parts of the code are the same.

- **bluepill-plus_swv_advanced**
This time, the `_write` function was overwritten again and the visibility of the code was somehow improved. LED blinking was also added before the delay to visually show the counting. The counting process can also be shown graphically. To do this, after switching to the Debug perspective, click on Window -> Show View -> SWV -> SWV Data Trace Timeline Graph. Next, click on the wrench icon in the SWV Data Trace Timeline Graph and, under the Comparator block, enter `count` into the 'Var/Addr:' field and check the 'Enable' box. This will allow you to use the already defined global static variable, i.e., `count`, to be monitored in the graphics form. Then, do the same as in point 5 of the [Software Setup](#software-setup) section above, but this time for the 'SWV Data Trace Timeline Graph' tab. You should see the step graph of the `count` variable values printed in the console.

> [!TIP]  
> To avoid losing the first character of the very first string being printed, a small delay of 0.1 seconds is inserted into the code before the start of the loop.

---

## References

1. https://www.st.com/resource/en/application_note/dm00354244-stm32-microcontroller-debug-toolbox-stmicroelectronics.pdf
2. https://sebastian.io/blog/stm32-swv-trace-debugging
3. https://lujji.github.io/blog/stlink-clone-trace/

