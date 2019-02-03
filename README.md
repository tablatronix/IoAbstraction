# IoAbstraction Arduino library summary

This library provides several useful extensions that make programming Arduino for non-trivial tasks simpler. There are many different practical and familiar examples packaged with it in the `examples` folder. Below I cover each of the main functions briefly with a link to more detailed documentation.

## Full API documentation

Along with ths quick start guide and the examples also see:

* [IoAbstraction documentation pages](https://www.thecoderscorner.com/products/arduino-libraries/io-abstraction/)
* [IoAbstraction reference documentation](https://www.thecoderscorner.com/ref-docs/ioabstraction/html)
* [TCC Libraries community discussion forum](https://www.thecoderscorner.com/jforum/)

## Installation for Arduino IDE

To install this library, simply download a zip (or source as preferred) and install into the `Arduino/libraries` directory, rename the library from IoAbstraction-master to IoAbstraction. Arduino sketches and libraries are normally stored under the Documents folder on most operating systems.

## TaskManager - simple, event based programming for Arduino 

Is a very simple scheduler that can be used to schedule things to happen either once or repeatedly in the future. Very similar to using setTimeout in Javascript or co-routine frameworks 
in other languages. It also simplifies interrupt handling such that you are not in an ISR when called back, meaning you can do everything exactly as normal. The only real restriction
with this library is not to call delay() or do any operations that block for more than a few microseconds. 

A simple example:

In the setup method, add an event that gets fired once in the future:

```
	taskManager.scheduleOnce(100, [] {
		// some work to be done.
	});
```
Then in the loop method you need to call: 

  	taskManager.runLoop();

### Advanced usage of TaskManager

If you want to improve task manager performance in code that frequently calls delayMicroseconds(..), you can enable the flag `_TASKMGR_OVERRIDE_DELAY_` by opening TaskManager.h
and reading the comment in the user definable section. At the moment, it's only enabled where I've personally tested it, SAMD (MKR, Zero etc) and AVR (Uno, Mega etc).

## BasicIoAbstraction - easily interchange between pins, PCF8574, MCP23017 and shift registers.

Lets you choose to use Arduino pins, shift register Input/Output, PCF8574 i2c and MCP23017 i2c in an inter-changable way. Use it in your sketch to treat shift registers or i2c expanders like pins. There's even an abstraction that can combine together Arduino pins and one or more other expander! See the documentation (link further up) for more details.

If you are building a library and want it to work with either Arduino pins, shift registers or an IO expander for IO, then this library is probably a good starting point.

A simple example:

If we want to use the i2c wire based ioFrom8574 we must include the wire header file

	#include <IoAbstractionWire.h>

At the global level (outside of any function) we create an i2c expander on address 0x20:

	IoAbstractionRef ioExpander = ioFrom8574(0x20);

Or for Arduino pins instead..
	
	IoAbstractionRef ioUsingArduino();

And lastly for DfRobot LCD shield input we use (requires library V1.3.2 at least):

	IoAbstractionRef inputFromDfRobotShield();   // for all other versions
	IoAbstractionRef inputFromDfRobotShieldV1(); // for version 1

In setup we set it's first IO pin to input and start the Wire library:
	
	Wire.begin();  
 	ioDevicePinMode(ioExpander, 0, INPUT);
  
And then later we read from it, in this case as we are doing a single read, use the 'S' version of the method as it removes the need to call the sync method. The only limitation is we must synchronize the device state. This allows us to be efficient where possible, setting several pins, syncing and then reading pins.

  	int valueRead = ioDeviceDigitalReadS(ioExpander, 0); // read pin 0 on ioExpander

Let's now say we wanted to write one value and read two items on the same device, in this case we don't use the 'S' version of the method, because otherwise it would sync three times.

	ioDeviceDigitalWrite(ioExpander, outputPin, HIGH);
	ioDeviceSync(ioExpander);
	int read1 = ioDeviceDigitalRead(ioExpander, inputPin1);
	int read2 = ioDeviceDigitalRead(ioExpander, inputPin2);

## SwitchInput - buttons that are debounced with event based callbacks

This class provides an event based approach to handling switches and rotary encoders. It full debounces switches before calling back your event handler and handles both repeat key and held down states. In the case of rotary encoders an interrupt on PIN_A is required, as the library needs to react very quickly; it is also important to make sure you have no long running tasks, or you'll miss the delayed rise. Note that this component also uses task manager.

Before doing anything else, you must add taskManager's run loop to your loop method, and your loop method must not do any long delay calls.

	void loop() {
		taskManager.runLoop();
	}

Here's a simple example example using a switch:

In setup we initialise it telling it to use arduino pins for IO, we could use shift registers or an i2c expander, and we also add a switch along with the event that should be:

	switches.initialise(ioUsingArduino(), pullUpLogic); // pull up logic is optional, defaults to PULL_DOWN buttons.
	switches.addSwitch(spinwheelClickPin, onClicked, NO_REPEAT); // NO_REPEAT is optional, sets the repeat interval in 100s of second.

Then we create a function for onClicked, this will be called when the button is pressed:

	void onClicked(uint8_t pin, bool heldDown) {
		// pin: the pin that was pressed
    		// heldDown: if the button has been held down
  	}

It is also possible to use initialiseInterrupt instead of initialise, when using this mode the library does not poll the switches unless a button is pressed down. It's use
is interchangable with initialise().

## RotaryEncoder - hardware and button emulation, even available with i2c IO expanders

Switch input also fully supports rotary encoders (and simulated rotary encoders using up / down buttons). For this you just initialise the rotary
encoder, but note that for rotary encoders PIN_A must be an interrupt pin, such as pin 2 on most boards. No debouncing is needed, the library
will switch on pull up resistors too, but you may need lower resistance pull ups will long wire runs.

For more see https://www.thecoderscorner.com/products/arduino-libraries/io-abstraction/arduino-switches-handled-as-events/.

	void onEncoderChange(int newValue) {
		// do something with new value..
	}

	// Example 1, Real encoder, we need to set up the pins that the encoder uses and provide a callback
	setupRotaryEncoderWithInterrupt(encoderAPin, encoderBPin, onEncoderChange);
	
	// Example 2, Up / down buttons acting like an encoder
	setupUpDownButtonEncoder(pinUpBtn, pinDownBtn, onEncoderChange);
	
	// After initialising, we set the maximum value (from 0) that the encoder represents
	// along with the current value
	switches.changeEncoderPrecision(maximumEncoderValue, 100);

## EepromAbstraction - support for both AVR and i2c AT24 EEPROMs with a common interface

The eeprom abstraction has several implementations, which makes it possible for libraries and code to be transparent from
AVR or AT24 based I2C eeprom storage, it even has a No-Op implementation as well. All the implementations shown below are interchangable
so if like me you switch between 8 and 32 bit boards, just change the EEPROM implementation!

### Avr Eeprom abstraction

This implementation uses the standard AVR EEPROM space for storage - only available on 8bit AVR such as Uno, MEGA.

To create an instance

	AvrEeprom avrEeprom;

### I2c AT24 EEPROM abstraction

A ground up implementation of the i2c eeprom protocol that should be compatible with the vast majority of i2c EEPROM devices.

To create an instance we pass the address of the chip (usually between 0x50 and 0x57), and also the page size, below are the page sizes 
for the most common devices. Consult the datasheet if unsure.

| ROM       | PageSize |
|-----------|----------|
| AT24C32   |       32 |
| AT24C64   |       32 |
| AT24C128  |       64 |
| AT24C256  |       64 |
| AT24C512  |      128 |


	I2cAt24Eeprom anEeprom(addressOfRom, pageSize);
 
Then during setup, you must ensure you call Wire.begin()

	void setup() {
		Wire.begin();
		
		// your other setup code.
	}
 
 
 ### NoEeprom - does nothing, but fulfills the interface.

Does nothing but implements the interface - useful sometimes..

To create an instance

	NoEeprom anEeprom;

 
### Reading and writing EEPROM values

Writing primitive values
 
	anEeprom.write8(romAddr, byteVal);
	anEeprom.write16(romAddr, value16);
	anEeprom.write32(romAddr, value32);

	byte by = anEeprom.read8(romStart);
	unsigned int i16 = anEeprom.read16(romStart);
	unsigned long i32 = anEeprom.read32(romStart);
	
Writing arrays and strings

	char data[20]; // example array to work with
	anEeprom.readIntoMemArray((unsigned char*)data, romStart, sizeof data);
	anEeprom.writeArrayToRom(romStart, (const unsigned char*)data, sizeof data);
	
## Other links

[https://www.thecoderscorner.com/electronics/microcontrollers/switches-inputs/basic-io-abstraction-library-pins-or-8574/]

