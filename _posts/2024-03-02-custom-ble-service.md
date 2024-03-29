---
layout: post
title: "GATT isn't just a trade agreement: creating a custom BLE Service"
tags: firmware,zephyr,ble
categories: 
---

The upside of selecting a widely used protocol, like BLE, is that there is a great deal of information and infrastructure surrounding it. The downside is that there is a great deal of information and infrastructure surrounding it. Some of this complexity is inherent, some is due to accomodating many divergent use cases, and some is just accumulated cruft, but all of it needs to be dealt with. 

<!--more-->

In a [previous update]({% link _posts/2024-02-21-wireless-control.md %}), I showed success with wireless control of the device, but using an existing service, the [Nordic LED Button Service](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/libraries/bluetooth_services/services/lbs.html). This worked, but the service is designed just for demonstration purposes, and we will need a lot more capability. We need our own service. There's a fair bit of detail required to do this, which I won't rehash, instead will just link [Nordic's course](https://academy.nordicsemi.com/courses/bluetooth-low-energy-fundamentals/). What's currently implemented is establishing a connection, turning PWM on and off, and reporting the measured frequency when polled. Below is a screenshot:

![screenshot](/assets/2024-03-02/wireless-freq-screenshot.png)

Under the readable characteristic, we see the value of the measured frequency, in millihertz. The input is a signal generator set to 42.0 Hz, so it seems to be working. The interface is not very pretty, it's a generic developer interface (Nordic's [nRF Connect app](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-mobile), on iOS), but it can be used for measurement and control. Code is [here](https://github.com/KalynaEnergy/mv/tree/v0.1.0), conveniently tagged for today's version. With some level of wireless interface in place, a measurement system, and control of a power stage, we are almost in a position where the rest of the system is just a [Simple Matter of Programming](https://en.wikipedia.org/wiki/Small_matter_of_programming). Not quite, there are a few other granular chunks to demonstrate, but getting closer.


