---
title: Get to Know RPi
date: 2013-10-20 22:00
layout: post
category: projects
comments: true
tags: arm RPi python circuit quadcopter 
description: "
As soon as my Raspberry pi (B) came on board, I could not wait to power it up. After several hours of installing operaing system, networking/environment configurations, the chip was ready to set out for future interesting projects, say, quadcopters.
" 
---

As soon as my Raspberry pi (B) came on board, I could not wait to power it up. I had downloaded the *Raspbian* image into my 16G Kinston class 10 SD card early the afternoon, so basically all things left were "clicking ok" actions. 

After the system was installed, I had to configure it to what I was used to.

#####Networking

The RPi only got two USB interfaces (from a USB hub internally) so I had to put SSH configuration on the highest priority, because the RJ45 en0 interface was malfunctioning with my Macbook. That was because my hostname contained an illegal character which I thought might be the '_' in **rpi_Obinna** (Obinna was a football player I used in PES2010 from Inter Millan). Anyway, I did not saw the error at the first glance so I had to use one USB itf (interface) for my EDUP wireless card and the other one switched between the keyboard and mouse. It wasn't long before **ssh-keygen** was configured and finished the configuration of networking.

<br />
#####VIM

The VIM pre-installed looked fine but colorschemes were not able to be recognized anywhere in the path shown by **:set runtimepath**. Therefore I had to **sudo apt-get purge vim** and **sudo apt-get install vim** to get my [vimrc]({{BASE_PATH}}/files/vimrc.txt) work properly.

<br />
#####Dev environment

The Raspbian made a good support for Python. That could be seen by its many pre-installed Python games and IDEs. Besides, it also installed python version from 2.7 to 3.2. But I prefer Python2.7 much more. Most APIs do not have a good compatibility with 3.0+ either. One line of Python script:

[![rpi_print]({{BASE_PATH}}/images/rpi_print.jpg)]({{BASE_PATH}}/images/rpi_print.jpg)


<br />
#####Quadcopter

The RPi board has provided GPIO pins, making I2C, SPI communications with other chips are possible. The first thing to make a Quadcopter is to have a steady flight control board, which requires sensors and flight control firmware. I have ordered an MPU6050 sensor chip (6-axis motiontracking sensor) and a breadboard to compose the hardware. I plan to plant the MWC (MultiWii Control) controller board program (based on AVR) to RPi (ARM based).

<br />
