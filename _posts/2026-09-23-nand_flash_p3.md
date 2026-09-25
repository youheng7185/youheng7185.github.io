# Reuse NAND Flash from SSD, Part 3

After soldering, I found out there is no signals after the logic shifter IC. So I go and recheck the schematics that I drew, and I actually wired up active low output enable pin to 3.3V. Then, I cut the trace and solder all OE to GND instead.

![oe_fail](https://github.com/youheng7185/youheng7185.github.io/blob/main/uploads/nand_p3/oe_pin.jpg?raw=true)

I am using a STM32 to bitbang the NAND interface this time, I don't know why my Pico C SDK has issue about delay_ms, it keep stucking in the delay loop, I can't get the timer to setup properly. I just fed up and use STM32 instead. The interface is same as previous SLC nand, but has DIR pin to control the logic shifter direction for DQx pins.

To read device id of the NAND, I send cmd 0x90 and address 0x00. It reply back with:
* 0x45
* 0x48
* 0x9a
* 0xb3
* 0x7e
* 0x6b
* 0x08
* 0x1e

After findings on the internet, I found this blog [https://ripitapart.com/tag/45-48-9a-b3/](https://ripitapart.com/tag/45-48-9a-b3/). Now, I can confirm its a TLC nand by SanDisk and follows Toggle. I am following this Toshiba TH58TFxxV23BAxx datasheet. It is the closest I could find and they shares Toggle and TLC properties.

So here is the summary of the flash:

* two CE, so 64GB per CE
* 16384 bytes + 1952 bytes spare area as a page
* 768 pages per block, but its only indexed as 256 pages, accessible with 0x01=LSB, 0x02=CSB, 0x03=MSB page (will explain later)
* I could not confirm how large is the block per plane and LUN yet

### Addressing

Previously SLC has 4 bytes of address, but now this TLC has 5 bytes of address.

* First and second bytes represent column, basically just index between a page, it uses two bytes for this, total 15 bits, maximum value should be 16384+1952-1, but normally reads and writes, we can just put all 0 as we start reading and writing at the beginning of a page.
* Third, fourth and fifth bytes represent row. Third byte address is basically addressing the address of a page inside a block. Fourth and fifth bytes shows the block number.

Seems simple right? But my nand flash can't do program odd or even block. I don't know why even until now.

### Block Erase

From the datasheet, block erase require us to send 0x60 command, 3 bytes of ROW address and a 0xD0 command. But after reading the datasheet with claude, the first byte of ROW address must be 0x00 because erase is by a block, so there is no need to send page address. 

### Page Read

Page read is different from SLC also. 

* Send 0x01=LSB, 0x02=CSB, 0x03=MSB as command
* Send command 0x00
* Send 5 byte address
* Send command 0x30
* Send command 0x05
* Send 5 byte address again
* Send 0xE0
* Read back 18336 bytes now

### Page Write

Page write now is quite different, as we know page write needs to program by page, and now the page actually is 3 smaller page. So each writes requires 18336*3 bytes of data. Start page write with LSB page at first.

* Send 0x01=LSB, 0x02=CSB, 0x03=MSB
* Send command 0x80
* Send 5 byte address
* Wait t_ADL
* Write 18336 bytes of data
* Repeat with CSB and MSB
* Send command 0x70 to start programming
* Wait for RB pin to go high

After testing the largest address, I could only get 4096 page to be usable, deducting about 400+ of initial bad blocks, mostly around page 3956 to 4096. (4096-400) * 16384 * 3 * 256 / 1024 / 1024 / 1024, I can get like 43GB of usable space from a 64GB nand, but currently I only can access odd page, so its just about 21GB.

Also, the BER on TLC is super high, normally the correct bytes are 99 percent only. So each read operation must go through ECC correction, LPC ECC can fix this. I haven't go through that part.

This NAND flash trial and error already spend me almost a week, I don't have much motivation to continue this, but I will definitely continue when I had the mood to do it.
