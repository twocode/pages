---
title: RPi GPIO support with BCM2835 Libarary
date: 2013-10-26 22:00
layout: post
category: projects
comments: true
tags: RPi arm c GPIO breadboard
description: "
Raspberry is an SoC with a Linux (Raspbian) installed. But it also provides GPIO (General Purpose Input/Output) pins like any other development boards. In order to communicate with other chips like MPU6050 or STM32 with I2C or SPI, it is necessary to have a test on its GPIO functionalities.
" 
---

Raspberry Pi has a BCM2780 CPU based on ARMv6 architecture:

	pi@raspberry ~/workspace/test/gpio $ cat /proc/cpuinfo
	Processor	: ARMv6-compatible processor rev 7 (v6l)
	BogoMIPS	: 697.95
	Features	: swp half thumb fastmult vfp edsp java tls 
	CPU implementer	: 0x41
	CPU architecture: 7
	CPU variant	: 0x0
	CPU part	: 0xb76
	CPU revision	: 7
	
	Hardware	: BCM2708
	Revision	: 000d
	Serial		: 000000007bed9c07

There is one C library for BCM2835 which is compatible with BCM2708. Only with the GPIO pins different. The library has made a quite good macro definitions to the 26 pins on P1 connector session on RPi. However, it's a little difficult to use as I used a GPIO adpater on breadboard with cables. The pins on the adapter was redefined:

[![gpio-adapter]({{BASE_PATH}}/images/rpi/gpio_adapter.jpg)]({{BASE_PATH}}/images/rpi/gpio_adapter.jpg)

so I have to make a set of definitions:

```c
#define PINP0 RPI_V2_GPIO_P1_11
#define PINP1 RPI_V2_GPIO_P1_12
#define PINP2 RPI_V2_GPIO_P1_13
#define PINP3 RPI_V2_GPIO_P1_15
#define PINP4 RPI_V2_GPIO_P1_16
#define PINP5 RPI_V2_GPIO_P1_18
#define PINP6 RPI_V2_GPIO_P1_22
#define PINP7 RPI_V2_GPIO_P1_07
#define PINPGND RPI_V2_GPIO_P1_06
#define PINCE1 RPI_V2_GPIO_P1_26
#define PINCE0 RPI_V2_GPIO_P1_24
#define PINSCLK RPI_V2_GPIO_P1_23
#define PINMISO RPI_V2_GPIO_P1_21
#define PINMOSI RPI_V2_GPIO_P1_19
#define PINRXD RPI_V2_GPIO_P1_10
#define PINTXD RPI_V2_GPIO_P1_08
#define PINSCL RPI_V2_GPIO_P1_05
#define PINSDA RPI_V2_GPIO_P1_03
```

based on this BCM Pins - Header table:

[![gpio-2]({{BASE_PATH}}/images/rpi/gpio_2.jpg)]({{BASE_PATH}}/images/rpi/gpio_2.jpg)

With the APIs provided, the lights could blink with a frequency of every 500ms. Test code is:

```c
#include <bcm2835.h>

#include "rpi_defs.h"

int main(int argc, char **argv)
{
    if (!bcm2835_init())
        return 1;
    bcm2835_gpio_fsel(PINP0, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINP1, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINP2, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINP3, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINP4, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINP5, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINP6, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINP7, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINCE1, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINCE0, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINSCLK, BCM2835_GPIO_FSEL_OUTP);
    bcm2835_gpio_fsel(PINMISO, BCM2835_GPIO_FSEL_OUTP);
    // Blink
    while (1)
    {
        bcm2835_gpio_write(PINP0, HIGH);
        bcm2835_gpio_write(PINP1, HIGH);
        bcm2835_gpio_write(PINP2, HIGH);
        bcm2835_gpio_write(PINP3, HIGH);
        bcm2835_gpio_write(PINP4, HIGH);
        bcm2835_gpio_write(PINP5, HIGH);
        bcm2835_gpio_write(PINP6, HIGH);
        bcm2835_gpio_write(PINP7, HIGH);
        bcm2835_gpio_write(PINCE1, HIGH);
        bcm2835_gpio_write(PINCE0, HIGH);
        bcm2835_gpio_write(PINSCLK, HIGH);
        bcm2835_gpio_write(PINMISO, HIGH);
        
        bcm2835_delay(500);
        
        bcm2835_gpio_write(PINP0, LOW);
        bcm2835_gpio_write(PINP1, LOW);
        bcm2835_gpio_write(PINP2, LOW);
        bcm2835_gpio_write(PINP3, LOW);
        bcm2835_gpio_write(PINP4, LOW);
        bcm2835_gpio_write(PINP5, LOW);
        bcm2835_gpio_write(PINP6, LOW);
        bcm2835_gpio_write(PINP7, LOW);
        bcm2835_gpio_write(PINCE1, LOW);
        bcm2835_gpio_write(PINCE0, LOW);
        bcm2835_gpio_write(PINSCLK, LOW);
        bcm2835_gpio_write(PINMISO, LOW);
        bcm2835_delay(500);
        
    }
    bcm2835_close();
    return 0;
}
```

<br />
