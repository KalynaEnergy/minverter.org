---
layout: post
title: Wireless control using BLE
tags: firmware
categories: 
---

A microinverter (or almost any piece of equipment) needs an interface to monitor and control its state. There are many ways this can be done: buttons and dials on the front panel, a separate wired port using some protocol, [power line communication](https://en.wikipedia.org/wiki/Power-line_communication) using the wired connections you already need, or radio. There are lots of advantages of RF communication: no physical connection needed, so you can cross walls and avoid wiring rat's nests; lots of bandwidth available; many many standards and low cost implementations available. There is a concern with interference from other RF sources, but we as a society have spent more than a century managing the issue. And not needing buttons on a front panel, and using the phone that everyone is carrying anyway, seems like a win. So let's see if that can work.

<!--more-->

For a protocol, I'm starting with [BLE](https://www.bluetooth.com/learn-about-bluetooth/tech-overview/). This is supported by the Nordic toolchain and Arduino board I'm using, and is an evolution of the ubiquitous Bluetooth ecosystem. Disadvantages include maybe limited range, but you can bump transmit power and use fancier antennas, BLE supports mesh networks allowing repeaters, and maybe you want to limit range for physical security reasons. 

In BLE, you need to define how the devices talk to each other. Eventually we will need our own BLE service, but for the moment Nordic's [BLE LBS service](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/libraries/bluetooth_services/services/lbs.html) seems adequate. It's set up for a simple demo allowing you to turn on and off an LED on the embedded device from an app on your phone. The documentation for the demo is [here](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/samples/bluetooth/peripheral_lbs/README.html), and the code is [here](https://github.com/nrfconnect/sdk-nrf/tree/main/samples/bluetooth/peripheral_lbs). It can be controlled with the [nRF Connect for Mobile](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-mobile) app, or the simpler [nRF Blinky app](https://www.nordicsemi.com/Products/Development-tools/nRF-Blinky). So all we need to do is stuff this sample code together with the PWM code, turning the output on and off using the BLE connection.

At this point I think it makes sense to have a proper repo for the project firmware. This now exists, as [KalynaEnergy/mv](https://github.com/KalynaEnergy/mv). I use a workqueue item to handle the power state changes, so it doesn't need to happen in the interrupt handler. There's some kind of issue with turning off the waveform timer then turning off the PWM pins. I suspect that sometimes if the waveform timer work is still executing, the turnoff can think it's done but the waveform timer still sets pins. The result is that it can get stuck. Inserting a delay works, but probably having a [mutex](https://academy.nordicsemi.com/courses/nrf-connect-sdk-fundamentals/lessons/lesson-8-thread-synchronization/topic/mutexes/) or some other sort of locking mechanism would be better. The whole system works: it turns on but doesn't start output, is visible to the app, shows when the app connects, turns on output when I move the slider on in the app, and turns it off again when I move the slider off. Kind of cool. And I think compliant with the demands of [IEEE 1547](https://standards.ieee.org/ieee/1547/5915/) 4.6, "shall be capable of responding to external inputs", "shall be capable of disabling the permit service setting and shall cease to energize". There's a lot of other things required to be standards compliant, but turning on and off is one of them. 