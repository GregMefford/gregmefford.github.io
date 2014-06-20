---
layout: post
title: "AVR, Meet Smart Card"
date: 2010-06-05 21:19:39 -0400
comments: true
categories: 
- AVR
- Microcontrollers
- Smart Cards
- Hardware
- C
---
As a part of my MS thesis research, I have a need to have precise control over an ACS ACOS5 Smart Card.
In this article, I will explain the code I have written to directly interface an Atmel ATtiny2313 microcontroller to an ACS ACOS5 cryptographic Smart Card.
Before developing this AVR C code, I experimented with various other methods of interfacing with a Smart Card at a low level, with varying amounts of success.
I will discuss some of the other methods in future articles, and outline how far I got with them and where there is still opportunity for improvement.

<!-- more -->

## Why a Microcontroller?

My requirement was to have a simple, controllable interface to a Smart Card, with minimal cost as a design goal.
I started out trying to modify a commercial Smart Card reader to allow protocol analysis while under control of a computer.
While I was successful in modifying the reader to provide the physical interface, the API for getting low-level access to the card turned out to be difficult.
I decided that if I had to work with the PCSC API at the lowest level, I might as well just work from the card data sheet and implement as much of the protocol as I needed in C code for my $2 microcontroller.
This helped me to meet my controllability requirement by having full control over each byte transmitted and recieved between the card and the microcontroller.
It also helped me to meet a low-cost design goal by using a very inexpensive microcontroller instead of an expensive commercial reader that has been modified, plus a computer to drive it.

## Getting Started with an AVR

To make prototyping simple, I bought a very helpful prototyping board for the Atmel ATtiny2313 microcontroller from [Evil Mad Scientist][ems].
This board is great because it provides silk-screened labels of each IO pin function and plenty of plated through-holes for soldering interface wires and simple components like power supplies, filter capacitors, and resonators.
For connecting to a Smart Card, I picked up a Smart Card breakout board from [SparkFun Electronics][sparkfun].
This breakout board offers a stanrdard physical interface to slide in a Smart Card and access its pins on plated through-holes, into which I mounted a row of standard single in-line pins that I could use for breadboarding and wiring up to the microcontroller breakout board.

[ems]: http://www.evilmadscientist.com/article.php/card2313
[sparkfun]: http://www.sparkfun.com/commerce/product_info.php?products_id=9440

To get a nice stable clock to the microcontroller (and ultimately to the Smart Card), I populated a 20 MHz ceramic resonator in the resonator footprint on the microcontroller breakout board.
To make use of this external clock, I calculated the appropriate bits using the ATtiny2313 data sheet and programmed the fuse bits as follows:

``` c Configuring ATtiny2313 to Use a 20 MHz External Resonator 
lfuse = 0xAF
hfuse = 0xDF
efuse = 0xFF
```

## Generating a Smart Card Clock

The first requirement of interfacing the microcontroller to a Smart Card is to provide the card with a stable clock between 1 MHz and 5 MHz.
Even though the data IO interface of the ACOS5 Smart Card is asynchronous, the card requires a continuous clock on the CLK pin in order to operate.
This was accomplished rather simply using the ATtiny2313's hardware PWM function.
The PWM generator in the ATtiny2313 offers a lot of advanced features, but to simply generate a clock at a given frequency and duty cycle, most of the more advanced options are not required.
To generate a 2 MHz clock with a 50% duty cycle, I used the following simple function, adapted from the example code given in the data sheet for using the PWM generator:

``` c Generate a 2 MHz Clock with 50% Duty Cycle 
void init_PWM_CLK(void) {
  // Generate a clock on OC0B pin.
  // Compare Output Mode:
  TCCR0A = (0 << COM0B1) | (1 << COM0B0) | // Toggle OC0B on Compare Match
           (1 << WGM01)  | (0 << WGM00);   // Clear Timer on Compare Match
  TCCR0B = (0 << CS02) | (0 << CS01) | (1 << CS00); // IO CLK, no pre-scalar
  OCR0A = 4; // Freq. = 20MHz / 2*(1+ORC0A) = 2MHz
}
```

I also made sure the data direction of the `OC0B` pin is set to output, so the clock will not default to a high-impedance input:

``` c Configure the OC0B Pin as an Output 
  int main(void) {
    DDRD = (1 << DDD5); // OC0B output pin
    init_PWM_CLK();
  }
```

## Answer to Reset

Now that we have a clock to drive the Smart Card, the next thing to do is to perform the appropriate power-up sequence to cause the card to reset.
When the card resets, it transmits an Answer To Reset (ATR) string.
This string of bytes is used by commercial readers to identify the specific card being interfaced with, and describes its capabilities.
In this case, I read the ATR with an oscilloscope just to make sure the card is properly resetting, but I do not need the AVR to do anything based on the ATR, since I know the capabilities of the ACOS5 card that I am using for my research.

Since I do not need to actually read the bytes of the ATR sequence, I just use the following start-up sequence:

``` c Smart Card Start-Up Sequence 
  // Power-up procedure
  set_vpp();
  _delay_ms(10);
  set_reset();
  init_USART_RX();
  _delay_ms(80);
```

The `set_vpp` and `set_reset` functions are just convenience methods to set the respective output pins high for the `VPP` and `RESET` pads on the Smart Card.
The `init_USART_RX` function is used to configure the built-in Universal Sychronous and Asynchronous serial Receiver and Transmitter (USART), which will be described in more detail in the following section.

## Data IO with USART

A Microcontroller is a powerful tool for this type of low-level system integration because these very inexpensive processors generally come with built-in hardware support for a wide variety of serial protocols like Pulse-Width Modulation (PWM), I2C, and SPI.
These can be implemented using the AVR's USART and Universial Serial Interface (USI) ports, which support varying word lengths, start bits, stop bits, parity-generation, and parity-checking.

### Configuring the USART Interface

For simplicity, the USART is configured to communicate at the default 9600 bps defined in the Smart Card standard.
I believe that the ACOS5 is also capable of communicating at 125 kbps, but I have not yet worked out the command to get it into that mode.
Both Answer to Reset (ATR) and the following communications will take place at 9600 bps unless configured to do otherwise.

The code to configure the USART interface is straightforward from the data sheet:

``` c Configure the USART to Communicate with the Smart Card 
  void init_USART_9600(void) {
    // Baud rate
    // UBRR = f_osc/(16*baud) - 1
    // For 9600Hz, UBRR ~= 129.2 = 20000000/(16*9600) - 1
    UBRRH = 0;
    UBRRL = 230; // 129 didn't look right on the oscilloscope. 230 works well.
    // Set frame format: 8data, 1stop bit, even parity
    UCSRC = (0 << USBS) | (3 << UCSZ0) | (1 << UPM1);
  }
```

Since a Smart Card only has a single IO pad, I implemented the AVR's transceiver interface by connecting the IO pad of the card to both the AVR's TX and RX pins.
The direction of data on the IO line can then be controlled by software by activating and deactivating the USART's transmit and receieve modes exclusively.
When in receive mode, both the TX and RX pin on the microcontroller become high-impedance inputs, allowing the Smart Card to transmit.
In transmit mode, the microcontroller drives the TX pin, sending data to the Smart Card.
The C code to configure receive and transmit modes is  straightforward, based on the example code provided in the data sheet.
Since the transmit and receive modes are both enabled in the same control register, setting one without setting the other implicitly disables the latter.
The only other notable thing is that the `TXC` bit should be set when entering transmit mode to prevent the USART hardware from transmitting whatever byte happens to be already in the transmit buffer.

The code is as follows:

``` c Configuring the USART for Transmit and Receive Modes 
  void init_USART_RX(void) {
    // Enable receiver
    UCSRB = (1 << RXEN);
  }
  void init_USART_TX(void) {
    // Enable transmitter and tell it the buffer is empty.
    UCSRB = (1 << TXEN) | (1 << TXC);
  }
```

### Transmitting and Receiving Bytes

The transmit and receive processes are also straightforward.
The only thing to keep in mind is that the actual USART hardware is asynchronous to the rest of the running code.
That is, once a byte is written to the output buffer, the code continues to execute while the byte is transmitted in the background at the specified baud rate.
To simplify the code structure and avoid using interrupt call-back routines, my code keeps itself sychronous by waiting for each transmitted character to be flushed from the buffer before proceeding.
Similarly, the code to receive a character blocks until a character is received.
In one sense, this is dangerous because it could wait forever if the card became unresponsive when the interface was expecing a byte.
In practice, this should not be much of a problem for me because my research will exercise the card in controlled and well-known routines.

The transmit and receive code is as follows:

``` c Transmitting and Receiving Data with the USART
  void USART_TX( unsigned char byte ) {
    // Clear the TX Complete flag by writing a 1 to it
    UCSRA = UCSRA | ( 1 << TXC );
    // Wait until transmit buffer is ready
    while( !( UCSRA & ( 1 << UDRE ) ) )
      ; // Spin-lock
    // Send the byte
    UDR = byte;
  }
  unsigned char USART_RX( void ) {
    // Wait until receive buffer is ready
    while( !( UCSRA & ( 1 << RXC ) ) )
      ; // Spin-lock
    // Send the byte
    return UDR;
  }
```

