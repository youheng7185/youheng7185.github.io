# Play around with eink price tag

# eink price tag, LG Innotek REBE-MZ21B

Content to test blog functionality, content is mirrored from (https://github.com/youheng7185/cc2530_eink)

## Main components on the pcb
* CC2530F256, 8051 based mcu with zigbee (https://www.ti.com/product/CC2530) (https://www.rhydolabz.com/documents/CC2530.pdf)
* WFT0213CZ16 LW 212x104, 2.13 inch E-Ink raw display, seems like its just a generic eink display with controller (https://www.waveshare.com/2.13inch-e-paper-c.htm)

## Steps to play around with it

* SDCC 8051 compiler instead of keil or paid iar workbench,

 (https://github.com/svn2github/sdcc/blob/master/sdcc/device/include/mcs51/cc2530.h)
 
 (https://sdcc.sourceforge.net/)
 
 (https://sourceforge.net/p/contiki/mailman/contiki-developers/thread/387146F7-366B-480B-9FB4-6F7A9F201E59@bristol.ac.uk/)
 
* Open source CC debugger implementation, (https://github.com/Larsjep/CCLib-ESP32)

## Hook up CC2530 to esp32s3

* solder up rst, data, clock, put a 33k resistor between the dd_o and dd_i. dd_i to data, dc to clock, rst to rst

```
int CC_RST   = 4;
int CC_DC    = 6;
int CC_DD_I  = 5;
int CC_DD_O  = 7;
```

```
$ py -3 cc_info.py -p COM5
INFO: Found a CC2530 chip on COM5

Chip information:
      Chip ID : 0xa524
   Flash size : 256 Kb
    Page size : 2 Kb
    SRAM size : 8 Kb
          USB : No

Device information:
 IEEE Address : 00124b0019b2
           PC : 0000

Debug status:
 [ ] CHIP_ERASE_BUSY
 [ ] PCON_IDLE
 [X] CPU_HALTED
 [ ] PM_ACTIVE
 [ ] HALT_STATUS
 [ ] DEBUG_LOCKED
 [X] OSCILLATOR_STABLE
 [ ] STACK_OVERFLOW

Debug config:
 [ ] SOFT_POWER_MODE
 [ ] TIMERS_OFF
 [X] DMA_PAUSE
 [X] TIMER_SUSPEND
```
 
* Dump flash failed, so apply the patch in the issues (https://github.com/Larsjep/CCLib-ESP32/issues/1), still dumped failed at 89 percent, I suspect its read protected

* Ignore dumping original firmware, now I can load new firmware compiled with sdcc

```
python3 cc_write_flash.py -p /dev/ttyACM0 --in=/home/lapchong/arm_mpu/cc2530/print/out/firmware.bin --erase
INFO: Found a CC2530 chip on /dev/ttyACM0

Chip information:
      Chip ID : 0xa524
   Flash size : 256 Kb
    Page size : 0 Kb
    SRAM size : 8 Kb
          USB : No
Sections in /home/lapchong/arm_mpu/cc2530/print/out/firmware.bin:

 Addr.    Size
-------- -------------
 0x0000   4018 B 

This is going to ERASE and REPROGRAM the chip. Are you sure? <y/N>:  y

Flashing:
 - Chip erase...
 - Flashing 1 memory blocks...
 -> 0x0000 : 4018 bytes 
    Progress 100%... OK

Completed
```

* The mcu would run the code a bit and stop, need to exit debug mode to let it run

```
python3 cc_resume.py -p /dev/ttyACM0
INFO: Found a CC2530 chip on /dev/ttyACM0

Chip information:
      Chip ID : 0xa524
   Flash size : 256 Kb
    Page size : 0 Kb
    SRAM size : 8 Kb
          USB : No
Exiting DEBUG mode...
CPU is now running
```

## Pinout

UART is using USART1 Alt. 2, SPI is using USART0 as SPI, got these value with multimeter in diode mode

| pin at package | pin function |
|---|---|
| P1_6 | UART_TX |
| P1_7 | UART_RX |
| P1_1 | EINK_RST |
| P1_2 | EINK_BUSY |
| P0_5 | EINK_CLK |
| P0_4 | EINK_CS |
| P0_3 | EINK_MOSI |
| P0_2 | EINK_DC |

## Steps to get display working

* get LLM to write init register code for uart, watchdog and spi
* To conclude up the pinout, uart is using usart1, alt 2, spi is on usart0, alt 1
* Found a exact same model eink display init code (https://github.com/kristianlm/atmega328-eink-heltec), but its weird as my display seems like a black and white only. Writing framebuffer after sending 0x13 command works, the 0x10 command don't do anything.
* The display seems like it's degraded, the contrast ratio is bad.
* Rpi pico is used as usb to uart converter, the esp32s3 is running the CCLib firmware and act as debugger without debugging features, only flashing.

## Image

![image](https://github.com/youheng7185/cc2530_eink/blob/main/eink.jpg)

| Priority apples | Second priority | Third priority |
|-------|--------|---------|
| ambrosia | gala | red delicious |
| pink lady | jazz | macintosh |
| honeycrisp | granny smith | fuji |

| pin at package | pin function |
|---|---|
| P1_6 | UART_TX |
| P1_7 | UART_RX |
| P1_1 | EINK_RST |
| P1_2 | EINK_BUSY |
| P0_5 | EINK_CLK |
| P0_4 | EINK_CS |
| P0_3 | EINK_MOSI |
| P0_2 | EINK_DC |