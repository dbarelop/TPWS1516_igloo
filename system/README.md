# Firmware working parts / protocol
Path to flash-file:

igloo/system/teamprojekt2/designer/impl1/top.pdb

Config:
* 9600 Baudrate
* 8 data bits
* 1 stop bit
* no parity
* no handshake

The CLK Pin of the Connector (main) component has to be connected to 1MHz clocksource.

## Knowing Bugs
* The oscillator for the AD Converter has a oscillation bug. In this bug-state the ADC can't convert any Voltage.
    * A quit fix of this Bug is: Disconnect all ports, connect at first the power supply and then the rest. 
* Caused on en Overvoltage Protection it isn't possible to convert negative Voltages.
    * If you know that you want to convert negative Voltages, you can change *GND* and *V+* on the chanel and convert the negative Voltage as a positive voltage.

## Peripherals

### LEDs
|  LED No. |  Indication  |
|----------|--------------|
| LED1     | Master busy  |
| LED2     | ADT7301 busy  |
| LED3     | EEPROM busy  |
| LED4     | AD7782 busy  |
| LED5     | Watchdog enabled  |
| LED8     | System alive (blinking)  |

### Switches
| Switch No. | Function   |
|------------|------------|
| 1          | En/disable watchdog for debug purposes |

\pagebrake

## Watchdog
If the watchdog is enabled and the busy-flag is active for more than
~130 ms, a software-reset for all components will be triggered.

The watchdog can be disabled via the switch 1 on the board. Make sure all jumpers are set correctly.


## Master
Device ID: 0b0000
### get firmware version
* can be used for connection pinging
* better distinction between firmware versions

#### TX
|              |
|--------------|
| 0x00   |
| get firmware version command |

#### RX
|         |                     |
|---------|---------------------|
| 0xAA    | 0bHHHHLLLL          |
| OK-byte | H-4 high-bits, L-4 low-bits, eg. 0b10000100 = version 8.4 |

## EEPROM
Device ID: 0b0001
### read

#### TX
|              |            |            |
|--------------|------------|------------|
| 0x10   | 0bxxxxxxxA | 0bAAAAAAAA |
| read command | x-don't care, A-address MSB  | A-address  |

#### RX
|         |                     |
|---------|---------------------|
| 0xAA    | 0bDDDDDDDD          |
| OK-byte | D-byte at address A |

### read 16bit

#### TX
|         |                     |
|---------|---------------------|
| 0x17    | 0bAAAAAAAA          |
| read 16bit command | A-address |

#### RX
|              |            |            |
|--------------|------------|------------|
| 0xAA   | 0bDDDDDDDD | 0bdddddddd |
| OK-byte | D-byte at address A  | d-byte at address A+1  |

### write
* must be handled with care: only 1 000 000 cycles endurance (should only be called by user, not automatically)
* writeprotection is disabled automatically triggered by reset

#### TX
|              |            |            |          |
|--------------|------------|------------|----------|
| 0x11         | 0bxxxxxxxA | 0bAAAAAAAA |0bDDDDDDDD|
| write command | x-don't care, A-address MSB | A-address  | D-byte   |

#### RX
|         |         |
|---------|---------|
| 0xAA    | 0xBB    |
| OK-byte | Done-byte |

### write 16bit
* must be handled with care: only 1 000 000 cycles endurance (should only be called by user, not automatically)
* writeprotection is disabled automatically triggered by reset

#### TX
|              |            |            |          |
|--------------|------------|------------|----------|
| 0x18         | 0bAAAAAAAA | 0bDDDDDDDD | 0bdddddddd |
| write 16bit command | A-address | D-byte  | d-byte for address A+1   |

#### RX
|         |         |
|---------|---------|
| 0xAA    | 0xBB    |
| OK-byte | Done-byte |

### erase all
* erases complete memory (all bits set to 1)
* must be handled with care: only 1 000 000 cycles endurance (should only be called by user, not automatically)
* eraseprotection is disabled automatically triggered by reset

#### TX
|              |
|--------------|
| 0x12   |
| erase all command |

#### RX
|         |         |
|---------|---------|
| 0xAA    | 0xBB    |
| OK-byte | Done-byte |


## AD Converter
Device ID: 0b0010

### HEX Code to Voltage calculation
* AIN 	//analog input (the real voltage you print out)
* VReff 	//refference voltage = 2.5
* rng		//range select
* N = 24 // number of bits got by ADC (MISO)
* dec_input //MISO (24 bit Vector) in Decimal

GAIN = 1 IF rng=2,56V ELSE 16;  
v = 1.024 * VReff;  
a = 2^(N-1);  
AIN = (v*((dec_input/a)-1))/GAIN;  


### read
* Reads all 24 bit seperated in three Bytes (from lowes to highest velued Byte(bit)).
* The 24 bit value is Signed!
* Highest bit '1': Indicates a zero or positive full-scale voltage.
* Highest bit '0': Indicates a negative full-scale voltage.

#### TX
|  	|
|-----|
|0x20|
|read command|

#### RX
|	 |   |   |   |
|---|---|---|---|
|0xAA|0bDDDDDDDD|0bDDDDDDDD|0bDDDDDDDD|
|OK-byte|last Highest Byte| second Byte| first lowest Byte|

### CH1
* Set the Chanal for the next AD Conversion on CH1.

#### TX
|   |
|---|
|0x23|
|chanel select command|

#### RX
|         |
|---------|
| 0xAA    |
| OK-byte |

### CH2
* Set the Chanal for the next AD Conversion on CH2.

#### TX
|   |
|---|
|0x24|
|chanel select command|

#### RX
|         |
|---------|
| 0xAA    |
| OK-byte |

### RNG1
* Set the range for the next AD Conversion on +- 2.56V

#### TX
|   |
|---|
|0x25|
|range select command|

#### RX
|         |
|---------|
| 0xAA    |
| OK-byte |

### RNG2
* Set the range for the next AD Conversion on +- 0.16V

#### TX
|   |
|---|
|0x26|
|range select command|

#### RX
|         |
|---------|
| 0xAA    |
| OK-byte |

## ADT Temperature sensor
Device ID: 0b0011

* temperature is updated on sensor every 1.5 sec

#### TX
|   |
|---|
|0x30|
| read temperature command |

#### RX
|              |            |            |
|--------------|------------|------------|
| 0xAA   | 0b00TTTTTT | 0btttttttt |
| OK-byte | T-temperature data (signed, MSB)  | t-temperature data (LSB)  |