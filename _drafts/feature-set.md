---
layout: post
title: Required features
tags: 
categories: 
--

To be a utilty interactive inverter, the system will need a number of things, here listed without worrying about overlaps and hierarchies:
  * power stage
  * boost stage
  * line filter
  * voltage and current measurement of input
  * voltage and current measurement of output
  * measurement of other levels as needed
  * temperature monitoring
  * frequency measurement
  * generate on board voltages for powering control
  * monitoring and control suitable for application
  * coordination with other units?
  * MPPT for PV input
  * anti-islanding 
  * Bluetooth or other comms for external monitoring/control, firmware update
  * external system for monitoring/control/update, phone app or webapp or equipment
  * I think IEEE 1547 demands a wire comms input? Maybe RS485 interface sufficient
  * indicator lights for development and troubleshooting
  * console connection for development
  * independent control loops for various subsystems
  * hardware PWM of wavetable