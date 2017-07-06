---
layout: post
title: "Screaming fast GPIO with Intel Galileo"
date: 2017-06-05 09:37:45 +0100
comments: true
categories: [x86, Intel, Galileo, Arduino, Linux]
---
{% img /images/galileo.jpg %}

Foreword
========
*Intel Galileo and the Quark X1000 SoC that power the Galileo, are projects on
which I was the leading Linux engineer. One problem we had when getting the SoC
up-and running as an Arduino host - was that the GPIOs didn't provide the set of
functionality that the Arudino base-libraries provided. For that reason a
<a href="http://www.cypress.com/file/37971/download">Cypress CY8C9520A I2C GPIO expander</a>
was added. The Cypress though is an I2C based device and correspondingly has
pretty disappointing GPIO performance - in comparsion to the base Arduino which
was able to toggle GPIOs in the MegaHertz range - we were achieving the hundreds
of KiloHertz.*

*Here is a post I made on the various different ways of driving GPIO on the
Galileo subsequent to the original release of the project in 2013.*

Screaming fast GPIO on Intel Galileo
====================================

Galileo - has two pins IO2 and IO3 through which we can drive significant data rates.
By default these two pins are routed to the <a
href="http://www.cypress.com/file/37971/download">Cypress</a>.


There are three methods to communicate with these pins - which have increasing throughput

digitalWrite()
--------------

digitalWrite(register uint8_t pin, register uint8_t val)

Using this method it is possible to toggle an individual pin in a tight loop @ about 477 kHz
```C
	pinMode(2, OUTPUT_FAST);
	pinMode(3, OUTPUT_FAST);
	digitalWrite(2, 1);
	digitalWrite(2, 0);
```
This is a read-modify-write operation meaning we first read the value of the
GPIO, then we modify the value, then we write the new value back.

This method isn't especially fast since we are ultimately going through GPIOLib
in user-space - which involves a context switch to kernel and back in order to
toggle a GPIO.

### Example digitalWrite() outputs 477kHz waveform on IO2:
```C
	setup(){
		pinMode(2, OUTPUT_FAST);
	}

	loop() {
		register int x = 0;

		while(1){
			digitalWrite(2, x);
			x =!x;
		}
	}
```

fastGpioDigitalWrite()
----------------------

fastGpioDigitalWrite(register uint8_t gpio, register uint8_t val)

This function actually lets you write directly to the registers - without going
through the code around digitalWrite() and consequently has better performance
than a straight digitalWrite(). Unlike digitalWrite() we won't context-switch
when updating the register associated with the GPIO. Only GPIO 2 and GPIO 3 are
supported using this method.

Using this method it is possible to toggle an individual pin (GPIO_FAST_IO2,
GPIO_FAST_IO3) at about 680 kHz.
```C
    pinMode(2, OUTPUT_FAST);
    pinMode(3, OUTPUT_FAST);
    fastGpioDigitalWrite(GPIO_FAST_IO2, 1);
    fastGpioDigitalWrite(GPIO_FAST_IO3, 0);
```
Again this uses read/modify/write - and can toggle one GPIO at a time

### Example fastGpioDigitalWrite - outputs 683kHz waveform on IO3:
```C
setup(){
	pinMode(3, OUTPUT_FAST);
}

loop() {

	register int x = 0;

	while(1){
		fastGpioDigitalWrite(GPIO_FAST_IO3, x);
		x =!x;
	}
}
```

fastGpioDigitalWriteDestructive()
---------------------------------
fastGpioDigitalWriteDestructive(register uint8_t gpio_mask)

Using this method it is possible to achieve 2.93 Mhz data toggle rate on IO2/IO3 individually or simultaneously
```C
    pinMode(2, OUTPUT_FAST);
    pinMode(3, OUTPUT_FAST);
```

It is the responsibility of the application to maintain the state of the GPIO
registers directly. To enable this a function called fastGpioDigitalLatch() is
provided - which allows the calling logic to latch the initial state of the
GPIOs - before updating later. This method just writes GPIO bits straight to the
GPIO register - i.e. a destructive write - for this reason it is approximately 2
x faster then read/modify/write. Twice as fast for a given write - means four
times faster for a given wave form - hence ~700kHz (680kHz) becomes ~2.8Mhz
(2.93 MHz)

### Example fastGpioDigitalWriteDestructive outputs 2.93MHz waveform on IO3:
```C
uint32_t latchValue;

setup(){

	pinMode(3, OUTPUT_FAST);
	latchValue = fastGpioDigitalLatch();

}

loop() {

	while(1){
		fastGpioDigitalWriteDestructive(latchValue);
		latchValue ^= GPIO_FAST_IO3;
	}

}
```

### Example-4 fastGpioDigitalWriteDestructive outputs 2.93MHz waveform on both IO2 and IO3:
``` C
uint32_t latchValue;

setup(){
	pinMode(2, OUPUT_FASTMODE);
	pinMode(3, OUPUT_FASTMODE);

	latchValue = fastGpioDigitalLatch(); // latch initial state

}

loop() {

	while(1) {

		fastGpioDigitalWriteDestructive(latchValue);
		if(latchValue & GPIO_FAST_IO3){

			latchValue |= GPIO_FAST_IO2;
			latchValue &= ~ GPIO_FAST_IO3;

		}else{
			latchValue |= GPIO_FAST_IO3;
			latchValue &= GPIO_FAST_IO2;

		}
	}
}
```
In other words the responsibility lies with the application designer in cases 3
and 4 to ensure the GPIO register values are correct - assuming - these values
matter to the application use-case.
