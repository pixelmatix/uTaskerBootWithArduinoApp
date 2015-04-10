Create an Arduino app that can be updated through the uTasker bootloader

## Bootloader details
The bootloader can be used to update firmware by replacing a file on the SD card, or by directly updating the Teensy's flash by exposing the application area of flash as a drive over USB-MSD.

By default, the loader first checks for a valid firmware file on the SD card.  If there is a valid file, it compares the file to the contents of the application area of flash, and overwrites the application area of flash with the file if the contents are different.  The file on the SD card is deleted after it is updated.  
TODO: what if the file is invalid or identical?

The USB-MSD method of updating firmware is intended to be used as a backup.  The bootloader appears as a USB-MSD drive only if there is no valid application found on the SD card or in application flash.  The USB-MSD method can be forced to start - skipping the SD card check - by grounding Teensy pin 32 or 16 when the board powers up.  When connected to a SmartMatrix panel, a single red LED on the panel blinks when in USB-MSD mode to indicate new firmware needs to be loaded.

The bootloader expects firmware in a specific format.  The file is a raw binary, sized exactly matching the application area in flash (from address 0x8080 - 0x3FFFF = 0x37F80 bytes).  The last two bytes of flash contain a CRC.  Any unused locations in flash should be filled with 0xFF.  The file must be named "software.bin".  
TODO: support wildcard filenames e.g. software*.bin?

This srec_cat command can convert a .hex file compiled with Arduino using the ["0x8080 Offset" linker script](https://github.com/pixelmatix/JumpToAppWithOffset) into a .bin file compatible with the bootloader:  
`srec_cat '(' Blink.cpp.hex -Intel -crop 0x8080 0x40000 -offset -0x8080 ')' -fill 0xFF 0x0000 0x37F7E -crc16-b-e 0x37f7E -xmodem -Output software.bin -Binary`

It takes a few seconds for the bootloader to check the SD card at reset before either starting a valid application or starting USB-MSD if there is no valid application.  If there is new valid firmware on the SD card, it can take over 10 seconds to update the firmware.  
Todo: add some visual feedback to see the current state of firmware

## USB-MSD Known Issues:
* OSX Path Finder (3rd party app) can't delete software.bin
	* Use OSX Finder to upgrade firmware
* OSX Finder - can't overwrite software.bin to upgrade firmware
	* delete software.bin file before copying new file over
* Idea for fixing both issues above: delete existing firmware on force boot?

## Pins used:

* Teensy pins 0/1 - RX/TX
* Teensy pin 32 (B18) or Teensy pin 16 (B0) - can be grounded to force USB-MSD mode
* Teensy pin 24 (A5) - status LED output blinks while the bootloader is running
* SPI pins matching SmartMatrix Configuration connect to an SD card
	* SPI_CS pin 15 (C0)
	* SPI_SCK pin 13 (C5)
	* SPI_MOSI pin 11 (C6)
	* SPI_MISO pin 12 (C7)
* SmartMatrix pins are set as output to drive panel

## Instructions for Aurora

### Creating Application .bin for bootloader
* Load Aurora sketch and compile with Linker script "0x8080 Offset".
* Navigate to the directory holding the .hex file that was generated (you can find this directory in the Arduino build console).
* Copy `uTaskerUsbMsd-SmartMatrix.hex` to this folder
* Run this srec_cat command to combine the two hex files, and create a .bin with CRC compatible with the bootloader:  
  `srec_cat '(' '(' Aurora.cpp.hex -Intel uTaskerUsbMsd-SmartMatrix.hex -Intel ')' -crop 0x8080 0x40000 -offset -0x8080 ')' -fill 0xFF 0x0000 0x37F7E -crc16-b-e 0x37f7E -xmodem -Output software.bin -Binary`

### Creating .hex file containing Bootloader, Application, USB-MSD
* Follow above instructions for "Creating Application .bin for bootloader"
* Copy `uTaskerBootloader-SmartMatrix.hex` to the build directory
* Run this srec_cat command to combine all three hex files:  
`srec_cat '(' '(' '(' Aurora.cpp.hex -Intel uTaskerUsbMsd-SmartMatrix.hex -Intel ')' -crop 0x8080 0x40000 -offset -0x8080 ')' -fill 0xFF 0x0000 0x37F7E -crc16-b-e 0x37f7E -xmodem -offset 0x8080 ')' uTaskerBootloader-SmartMatrix.hex -Intel -Output AuroraWithBootAndUsbMsd.hex -Intel`
* Note this file can't be opened in Teensy Loader (it appears to have a bug where it won't accept a file that fills the flash)

### Loading bootloader and application to Teensy
* Create software.bin
* Use Teensy Loader to upload `uTaskerBootloader-SmartMatrix.hex` to Teensy
* Wait for red light to blink on SmartMatrix indicating no application is loaded.
* Find UPLOAD_DISK drive
* Copy software.bin to UPLOAD_DISK drive, it will eject automatically
* Ignore any warnings on "Disk Not Ejected Properly", the disk self-ejected after the file copied over
* Aurora should now run

### uTasker Compilation/License
uTaskerBootloader-SmartMatrix.hex is the uTaskerSerialLoader application included with V1.4.7 with some significant modifications to the .bin file format and logic to combine the USB-MSD and SD loaders.

The binary was compiled with Kinetis Design Studio.
A uTasker license was purchased for this project.  In keeping with the terms of the uTasker license, this binary can only be used for a non-commercial project.  If you want to use this project for a commercial project, uTasker has [reasonable license fees](http://www.utasker.com/Licensing/License.html) and excellent support.  You should compile the project yourself to customize the USB IDs and device information.

I do plan on open sourcing the changes I made to the uTaskerSerialLoader application, most likely as a diff that can be used with the uTasker V1.4.7 release published on uTasker's site.
