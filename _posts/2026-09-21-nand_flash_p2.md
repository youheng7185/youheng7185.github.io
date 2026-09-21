# Reuse NAND Flash from SSD, Part 2

Let's continue the journey from Part 1. After I recevied the PCB, I repaste the NAND ic with BGA132 stencils and solder everything together.

But before trying the new flash, I had some NAND flash breakout board before this, they are old SLC NAND. Probably you can find these in industrial devices or USB flash drive from maybe 20 years ago. I bitbang the NAND interface with a RP2040 just to build up my mental model about nand flash. They are W29N01HVS1NA 128MB SLC and MT29F2G08ABAEAWP 256MB SLC.

![onfi1_flash](https://github.com/youheng7185/youheng7185.github.io/blob/main/uploads/nand_p2/onfi1.jpg?raw=true)

Here is the explanation of the pins function on these NANDs, they are the basic sets of pins, newer flash are also using these but added a few for synchronous interface and strobe like on DDR SRAM.

* ONFI 1.0 standard, everything is asynchronous, so there is no CLK signal like I2C or SPI, the most common asynchronous protocol would be UART
* DQ0 to DQ7, 8 bit bidirectional
* RE_n, read enable, active low
* WE_n, write enable active low
* ALE, address latch enable, active high
* CLE, command latch enable, active high
* CE_n, chip select, active low
* WP_n, write protected, active low
* RB_n, ready busy signal, if nand flash is busy, its 0, this is an open drain io, so I added pull up resistor

To simplify the model, always just put CE_n as low and WP_n as high, so it makes the flash active and will execute our write commands.

To do read, write and erase operations, they are actually build up from these smaller operations:

### Send Command

* Set DQ0 to DQ7 as output mode, put the command on DQ0 to DQ7, DQ0 is the lowest bit
* Set CLE to high to latch the command
* Toggle WE_n from high to low, then low to high to complete send command operations

### Send Address

* Set DQ0 to DQ7 as output mode, put the address on DQ0 to DQ7
* Set ALE to high to latch the command
* Toggle WE_n from high to low, then low to high

### Send Data

* Set DQ0 to DQ7 as output mode, put the address on DQ0 to DQ7
* Toggle WE_n from high to low, then low to high

### Read Data

* Set DQ0 to DQ7 as input mode
* Toggle RE_n from high to low
* Wait a short period of time, samples DQ0 to DQ7
* Toggle RE_n from low to high, and continue to toggle RE_n from high to low if we need another byte of data

After understanding these fundamentals operations, we can do read, erase and write operation. These are basically all the functions needed to be able to use a NAND flash. These infos are from W29N01HV datasheet.

### Read Parameter Page

To verify the electrical connections, we always either read parameter page or read id. I prefer parameter page because its longer.

* Send command 0xEC
* Send address 0x00
* Wait for flash ready signal
* Read back 256 bytes

Oh, forgot to explain NAND flash fundamentals here, these info are under Memory Array Organisation in the datasheet.

* There is SLC, MLC, TLC and QLC NAND flash, SLC means 1 bit per cell, MLC 2 bit per cell, TLC 3 bit per cell and QLC is storing 4 bit per cell.
* The data that we write doesn't represent the voltage level in the cell, these engineers uses grey code and store them in the cell from the data we provided
* Classification from smallest to the largest, page < block < plane < LUN (logical unit number) < target (number of CE)
* Erase must by a block, write must by a page, this is not some SRAM where you can just erase a byte and write a byte
* Block on this SLC NAND is 2048+64 bytes, the extra 64 bytes is for us to store ECC data
* NAND flash requires ECC to protect the data from corruption, SLC uses simpler ECC algorithm while QLC and MLC requires much stronger algorithm. Because the BER (bit error rate) is much higher on QLC.

### Page Read

* Send command 0x00
* Send 4 bytes of address
* Send command 0x30
* Wait for flash ready signal
* Read back maximum 2048+64 bytes of data

### Block Erase

* Send command 0x60
* Send 2 bytes of address (because we are erasing by block, the lower two bytes of address are indexing the pages in a block)
* Send command 0xD0
* Wait for flash ready signal
* Send command 0x70 to read status
* Read one byte of data to check erase status

### Page Program

* Send command 0x80
* Send 4 bytes of address
* Send all the 2048+64 bytes of data to be latched into the data register
* Send command 0x10 to start program
* Send command 0x70 to program status
* Read one byte of data to check program status

There is also Random Data Output which people used to skip the 2048 bytes of data and read directly starting from the 64 bytes data region. Another important thing from the datasheet, there would be initial invalid blocks when the NAND flash is shipped from the factory, so NAND flash controller usually would read spare area to check the invalid blocks when it is in manufacturing or production mode of the SSD.

Enough for now, I also got basic read, erase and write working on the NAND scrapped from the SSD. Gonna explain them in the next part of this series.