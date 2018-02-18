# AVR8 GPIO configurability, made nice

Ok, so you've written C for 8 bit AVRs and some times you've had to port
your code to a different MCU in the same family, once that has happened
a couple of times you notice that you're writing a board.h file with
something like this:

```
#define LED_PORT PORTB
#define LED_DDR DDRB
#define LED_PIN PB0
```

This gives three or four (if you need to read from the pin as well) constants
for each GPIO pin you need to be able to move around.

So your code ends up looking like this:
```
void init() {
     LED_DDR |= _BV(LED_PIN);
}

void setLed(uint8_t on) {
    if (on) {
       LED_PORT |= _BV(LED_PIN);
    } else {
       LED_PORT &=~ _BV(LED_PIN);
    }
}
```

After a while you start to hate yourself for writing such code, but what
if there was a better way? Enter avr8gpio.h which in stead allows you to say:

```
#include "avr8gpio.h"

#define LED GPB0

void init() {
    GPOUTPUT(LED);
}

void setLed(uint8_t on) {
    GPWRITE(LED, on);
}

```

Now, you might object that this is bound to be hideously inefficient like
the digitalWrite function in the Arduino libraries, but you'd be wrong because
the code generated when using this header is identical to what's generated
when using the traditional avr/io.h header.


# What you get

In these exmples x is the port name (A, B, C, D etc.) and y is the pin number
in the port (0,1,2,3,4,5,6,7):

## A constant for each GPIO pin which contains both the port and the pin

The header defines a GPxy symbol for Port x and pin number y for each available
GPIO pin on the particular processor, iow. PB0 gets a GPB0 constant which
identifies both PORT B and bit 0.

Only the pins that actually exist get a GPxy constant, so you should get the
same compile-time validation that you're used to.


## Macros for using the GPxy constants

Some times you need raw access to the GPIO ports, traditionally you'd use
DDRB to set the direction of pins of port B, but with avr8gpio.h you have a
macro called GPDDR which takes a pin constant, so to get at the direction
register for a pin you just say GPDDR(GPB0), the same is true with PINx and
PORTx.

For most cases, you'd probably want to use the more convenient macros for
setting direction of a pin (GPOUTPUT or GPINPUT), setting or clearing
an output bit (GPSET, GPCLEAR, GPWRITE) or reading a pin (GPREAD).


### PINx => GPPIN(GPxy)
```
// Equivalent to PINA, PINB, etc. so PINA is the same as GPPIN(GPA0)
#define GPPIN(port) (*(volatile uint8_t *)((port) >> 3))
```

### DDRx => GPDDR(GPxy)

```
// Equivalent to DDRA, DDRB, etc. so DDRB is the same as GPDDR(GPB7) 
#define GPDDR(port) (*(volatile uint8_t *)(((port) >> 3)+1))
```

### PORTx => GPPORT(GPxy)
```
// Equivalent to PORTA, PORTB, etc. so PORTC is the same as GPPORT(GPC3)
#define GPPORT(port) (*(volatile uint8_t *)(((port) >> 3)+2))
```

### _BV(Pxy) => GPBV(GPxy)
```
// The bit value of the pin part
#define GPBV(port) (1 << ((port) & 7))
```

### DDRx |= _BV(Pxy) => GPOUTPUT(GPxy)
```
// Configure the pin as output DDRA |= _BV(PA0) is the same as GPOUTPUT(GPA0)
#define GPOUTPUT(port) GPDDR(port) |= GPBV(port)
```

### DDRx &=~ _BV(Pxy) => GPINPUT(GPxy)
```
// Configure the pin as input DDRA &=~ _BV(PA0) is the same as GPINPUT(GPA0)
#define GPINPUT(port)  GPDDR(port) &=~ GPBV(port)
```

### PORTx |= _BV(Pxy) => GPSET(GPxy)
```
// Set the output bit PORTA |= _BV(PA0) is the same as GPSET(GPA0)
#define GPSET(port) GPPORT(port) |= GPBV(port)
```

### PORTx &=~ _BV(Pxy) => GPCLEAR(GPxy)
```
// Clear the output bit PORTA &=~ _BV(PA0) is the same as GPCLEAR(GPA0)
#define GPCLEAR(port) GPPORT(port) &=~ GPBV(port)
```

### Writing a specific output bit => GPWRITE(GPxy, state)
```
// Write a bit to either 1 or 0
#define GPWRITE(port, on) if (on) {GPSET(port);} else {GPCLEAR(port);}
```

### Reading a specific input bit => GPREAD(GPxy)
```
// Read an input pin PINA & _BV(PA1) is the same as GPREAD(GPA1)
#define GPREAD(port)  (GPPIN(port) & GPBV(port))
```


# Abstracting GPIO pins and ports for more readable code

The entire point of avr8gpio.h is to make it tidy to define constants for
pointing out GPIO pins in a central location of the code. My projects
contain a file called board.h which might look something like this:

'''
#pragma once
#include "avr8gpio.h"

#if BOARD_VERSION==1

  #define LED GPB2
  #define SENSOR GPB0

#elif BOARD_VERSION==2

  #define LED GPA7
  #define SENSOR GPF3

#else
  #error BOARD_VERSION not defined
#endif

'''

This means that my code never has direct references to the GPxy constants
outside the board.h file, so it's easy to see what the code is actually doing.

