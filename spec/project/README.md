# Project specification

This document captures the top level project specifications. Other component level specifications will strive to achieve the objectives defined in this specification

# Overview

timon is a system that allows you to generate digital functions (a.k.a "wave forms") in its channels based on a set of primitives (a.k.a commands) that are passed on to the device and "read" the data present in the device channels. Additionally , it will have simple interface functionalities like SPI,I2C and UART that are also configurable and controllable using the interface software.

The primary components of timon are:

- Hardware
- Firmware
- Software

# Functionality

The following functionality are supported by timon

- DIO mode  :  here timon acts as a digital I/O device that can generate a function in its channels as well as read serial data input from its channels.
- UART mode :  here timon will act as a USB-UART bridge device
- SPI mode  :  timon acts as a USB-SPI bridge in this mode
- I2C mode  :  timon acts as a USB-I2C bridge in this mode.
- PWM       :  use timon to generate a squarewave of desired frequency and duty cycle
 
# Specification


## Channels

timon has 8 digital channels named `7-0`. These channels can be configured as input, output or a functional channel. Example of a functional channel is setting the channel to be SDI and SDO pins in SPI mode. Not all channels can be mapped to all functions. The mapping possibility will be defined in the corresponding mode specification.

|     |     |     |     |     |     |     |     |
| --- | --- | --- | --- | --- | --- | --- | --- |
|   7 |   6 |   5 |   4 |   3 |   2 |   1 | 0   |

## Communication Interface

timon interfaces with the interface software using a High speed USB interface. It enumerates as a USB-CDC device and will be visible to the software as a serial port running with 921600 8N1 mode with flow control enabled.

## Commands

The device interacts with the interface software using commands. Each command has a command name (mnemonic) and an opCode and can have zero or more argument.

Since the interface enumerates as a serial port, all data transfer happens in the form of ASCII encoded characters. The software and firmware should do required conversions into the defined data formats before processing the command.

All command opCodes are ASCII encoded string of three character length. For instance the setMode command is encoded as the string "001". The device should convert this into a 32bit unsigned integer to start operating on it.

Arguments if any will be from the 4th character in the string. Length of the argument is command specific.  

Command strings (opCode + arguments) should be terminated with a "\r"


## acknowledgement

When the device is done with performing a command and is ready to receive the next command, it will send a prompt indicator ( `>` [ASCII: 0x3E] ) in a new line as a CTS indication.

If the previous command failed , a failure string followed by an empty prompt indicator with a failure indicator ( `!` [ASCII: 0x21] ) will be sent. This will be followed by a regular prompt indicator. All these will be separated by `\r` .

For example:

```
Failed to process command. invalid argument.
>!
>
```

## Setup

To set the hardware into the required mode, the interface software can use the `setMode` command as described below. The mode specific settings and default configuration of each mode will be defined in the corresponding mode specification sections.

### setMode command

|           |               |
|-----------|---------------|
| mnemonic  | setMode       |
| opCode    | 001           |
| arguments | DIO mode (01) |
| 	    | UART mode (02)|
|	    | SPI mode (03) |
| 	    | I2C mode (04) |

_Usage sample:_ the command string "00101\r" will set the device to DIO mode and return `>` to indicate operation has been successfully performed.    

### statCheck

statCheck is a special single character command that can be used to synchronize the device and software operation. On receiving the statusCheck command, the device will respond with a CTS symbol [`>`] when ready.

|           |                       |
|-----------|---------------        |
| mnemonic  | statusCheck           |
| opCode    | `\r` [ASCII: 0x0D]    |
| arguments | DIO mode (01)         |


## DIO mode

In DIO mode, the device operates as a USB-I/O bridge. It listens to commands and drives / since signals into the device channels.

### General operation

Following are the DIO mode commands:

- dioSetTick
- dioSetDir
- dioGetDir
- dioRead
- dioWrite
- dioExec
- dioStopExec
- dioResetExec
- dioFlushExec
- dioFlushRb
- dioFetch

The steps to be taken to operate in DIO mode are :

- set the execution frequency using `dioSetTick`
- set the channel directions using `dioSetDir`
- fill the device ring buffer with commands
	- a buffer command can either be a read from multiple channels or a write to multiple channels only.
- issue the execute command `dioExec`
- issue the fetch command  `dioFetch`
	-  This will make the device send back data from any channels that has been setup to read data.

Device will store all read and write commands in ring buffer and executes on command. The ring buffer size is 512 bytes. Data read in from the channels will be de-serialized into 32 bit words and stored in channel specific ring buffers of 64 words each. This will be transmitted to the interface software when the fetch command is issued.

### dioSetTick

|           |                                    |
|-----------|---------------                     |
| mnemonic  | dioSetTick                         |
| opCode    | 002                                |
| arguments | operating period in  microSeconds  |

The device will setup an internal timer and execute each instruction in a ring buffer every time the timer expiry interrupt occurs. This timer frequency can be set using the `dioSetTick` command.

_Usage Sample:_ `002100\r` will set the tick period to 100us

### dioSetDir

|           |                                                   |
|-----------|---------------                                    |
| mnemonic  | dioSetDir                                         |
| opCode    | 003                                               |
| arguments | 2 char hex representation of direction bitmap |

To set the direction of the channels , provide a three digit hex representation of the bitmap of the directions of all channels in the device in ASCII. `0` denotes output and `1` denotes input. As soon as DIO state is entered, all channels will be configured as output.  

_Usage Sample:_ to set direction of channel 1 to input, issue `00302\r`

### dioGetDir

|           |               |
|-----------|---------------|
| mnemonic  | dioGetDir     |
| opCode    | 004           |
| arguments | N/A           |

Read out the current direction settings of all channels.  Device will return the 3 char hex representation of direction bitmap from the device.

_Usage Sample:_ on successfully processing the command `004\r`, the device will return the current direction bitmap in the sample form `002\r`

### dioSetPullUp

|           |               |
|-----------|---------------|
| mnemonic  | dioSetPullUp  |
| opCode    | 013           |
| arguments | N/A           |


To enable the channel pullups , provide a three digit hex representation of the bitmap of the directions of all channels in the device in ASCII. `0` denotes pullup disabled and `1` denotes pull up enabled. As soon as DIO state is entered, all channels will be configured as pullup disabled.  

*Note* If a pullDown is already set in the channel, this command should return an error. 

_Usage Sample:_ to enable the pullup of  of channel 1 , issue `01302\r`


### dioSetPullDn

|           |               |
|-----------|---------------|
| mnemonic  | dioSetPullDn  |
| opCode    | 014           |
| arguments | N/A           |


To enable the channel pulldown , provide a three digit hex representation of the bitmap of the directions of all channels in the device in ASCII. `0` denotes pulldown disabled and `1` denotes pulldown enabled. As soon as DIO state is entered, all channels will be configured as pulldown disabled.  

*Note* If a pullup is already set in the channel, this command should return an error. 

_Usage Sample:_ to enable the pulldown of  of channel 1 , issue `01402\r`

### dioGetPullUp

|           |               |
|-----------|---------------|
| mnemonic  | dioGetPullUp  |
| opCode    | 015           |
| arguments | N/A           |

Read out the current pullUp settings of all channels.  Device will return the 3 char hex representation of pullUp bitmap from the device.

_Usage Sample:_ on successfully processing the command `015\r`, the device will return the current pullUp status bitmap in the sample form `001\r`

### dioGetPullDn

|           |               |
|-----------|---------------|
| mnemonic  | dioGetPullDn  |
| opCode    | 016           |
| arguments | N/A           |

Read out the current pullDown settings of all channels.  Device will return the 3 char hex representation of pullDown bitmap from the device.

_Usage Sample:_ on successfully processing the command `016\r`, the device will return the current pullUp status bitmap in the sample form `001\r`


### dioWrite

|           |               |
|-----------|---------------|
| mnemonic  | dioWrite      |
| opCode    | 005           |
| arguments | 2 digit character representation of hex of bitmap to be written into the channels in ASCII |

The `dioWrite` command will store the bitmap in the command into the ring buffer storing commands for execution. Channels set as inputs should have data `0` corresponding to it.   

_Usage Sample:_ `00501\r` will insert the command to set channel 0 high into the command ring buffer.

### dioRead

|           |               |
|-----------|---------------|
| mnemonic  | dioRead       |
| opCode    | 006           |
| arguments | 2 digit character representation of hex of bitmap channels to be read in ASCII |

To read data from a set of channels , issue the `dioRead` command with the hex string representation of the bitmap of channels to be read.

_Usage Sample:_ `00603\r` will read the channels `0` and `1` and shift the data into the corresponding channel read ring buffers. Once 32 bits have been shifted into the register, the pointer is incremented into the next channel read buffer in the channel read ring buffer set.  

### dioExec

|           |                                 |
|-----------|---------------                  |
| mnemonic  | dioExec                         |
| opCode    | 007                             |
| arguments | execution count word in hex string.  |

`dioExec` will execute the commands in the command ring buffer for the number of times mentioned in the argument. The execution count is a word long and `0` indicates to continue execution until `dioStopExec` is issued.

_Usage Sample:_ `007F3F7` will cyclically execute the instructions in the ring buffer 62455 times.

### dioStopExec

|           |              |
|-----------|--------------|
| mnemonic  | dioStopExec  |
| opCode    | 008          |
| arguments | N/A          |

Stop current execution if any.

_Usage Sample:_ `008\r` will stop current execution if any

### dioResetExec

|           |              |
|-----------|--------------|
| mnemonic  | dioResetExec  |
| opCode    | 009          |
| arguments | N/A          |

Reset execution pointer to start of execution ring buffer

_Usage Sample:_ `009\r` will reset execution pointer to start of the execution ring buffer

### dioFlushExec

|           |              |
|-----------|--------------|
| mnemonic  | dioFlushExec  |
| opCode    | 010          |
| arguments | N/A          |

Invalidate the execution ring buffer.

_Usage Sample:_ `010\r` will reset execution ring buffer

### dioFlushRb

|           |                                      |
|-----------|--------------                        |
| mnemonic  | dioFlushRb                           |
| opCode    | 011                                  |
| arguments | 2 byte hex string of bit map of read budders to flush  |

Invalidate contents of all read ring buffers mentioned in the bitmap

_Usage Sample:_ `011FF\r` will clear and reset all read buffers.


### dioFetch

|           |                                             |
|-----------|--------------                               |
| mnemonic  | dioFetch                                    |
| opCode    | 012                                         |
| arguments | 1 byte hex string of read buffers to fetch  |

Pass contents of the read ring buffer to the software as a \r delimited stream of hex encoded data.

_Usage Sample:_ `0127\r` will transfer contents of read ring buffer 7 to the software.





# Command summary

| mnemonic      | opCode        | Arguments                                |
|-----------    |---------------|------------------                        |
| statCheck     | `\r`          | N/A                                      |
|               |               |                                          |
| setMode       | 001           | -  DIO mode (01)                         |
| 	        |               | - UART mode (02)                         |
|               |               | - SPI mode (03)                          |
| 	        |               | - I2C mode (04)                          |
| dioSetTick    | 002           | tick period in us                        |
| dioSetDir     | 003           | 2 character direction hex as string      |
| dioGetDir     | 004           |N/A                                       |
| dioWrite      | 005           | 2 character write data  as hex           |
| dioRead       | 006           | hex string of channels to be read        |
| dioExec       | 007           | hex string of execution count            |
| dioStopExec   | 008           |N/A                                       |
| dioResetExec  | 009           |N/A                                       |
| dioFlushExec  | 010           |N/A                                       |
| dioFlushRb    | 011           |2 character hex string of Rbs to flush    |
| dioRead       | 012           |1 character hex string of Rb to read      |
| dioSetPullUp  | 013           |2 character pullUp bitmap hex as string   |
| dioSetPullDn  | 014           |1 character pullDown bitmap hex as string |
| dioGetPullUp  | 015           |N/A                                       |
| dioGetPullDn  | 016           |N/A                                       |
