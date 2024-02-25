---
layout: post
title: Correct use of the Zephyr system workqueue, aka multiprogramming is fun
tags: firmware,zephyr
categories: 
---

[Last time]({% link _posts/2024-02-21-wireless-control.md %}) I mentioned an issue with the output sometimes getting "stuck" rather than being turned off correctly. Having a good understanding of how and when things happen in the control system is important, particularly if what is being controlled is safety critical. And power conversion equipment is. So what was happening?

<!--more-->

The system has a timer set up, to periodically update the PWM output level in accordance with the desired waveform. This is done in a handler. The system also has wireless control setup, over BLE, which turns on and off the timer, and sets all the pins to zero. Sometimes, after the system had been turned off, and pins zeroed, they sometimes got turned on again, and stuck there. Not good. No amount of timer synchronization, or inserted delays, was able to fix the issue. What was happening is that all the handlers, for both doing step increments and dealing with commands from Bluetooth, was work in the [Zephyr system workqueue](https://docs.zephyrproject.org/latest/kernel/services/threads/workqueue.html). So you can have the following sequence of events:

*  BT module receives turn off command via Nordic LBS service. 
*  BT ISR statechange_cb invoked.
	* Submits statechange work to system workqueue
	* Leaves ISR

At this point, system workqueue contains an item to do the statechange, but hasn’t begun executing it yet

*  Step_timer expires
*  Timer ISR step_timer_handler gets called
    * Step_timer_handler submits step_work to the system workqueue
	* Leaves ISR

System workqueue now also contains a step_work item, after the statechange item, but hasn’t begun executing it yet either

* Now system workqueue thread wakes up, and starts processing items.
* First is statechange_handler, which calls trip_off()
* In trip_off:
	* Step_timer is stopped
	* Timer is synced, which should allow any further timer expiries to trigger
	* A delay is added, and some logging
	* Only then do we turn off all the PWM
	* Returns

Next, system workqueue still has a step_work item, which has been waiting all this time, and begins processing it by calling step_handler

*  Step_handler:
    * Sets PWM to the state it was expecting, turning PWM lines on again
    * Returns

The timer is stopped, so this condition remains. No amount of timer synchronizing or delays in statechange_handler will fix it, because the problem lies waiting in the next item in the workqueue. The workqueue items are processed completely, first in, first out. Serialized, if you will. So what to do?

One could try to clean items out of the workqueue during turnoff, cancel them or something. But if I understand [best practice](https://docs.zephyrproject.org/latest/kernel/services/threads/workqueue.html#workqueue-best-practices) correctly, we should just check in the handler whether to actually do the work:

> A general best practice is to always maintain in shared state some condition that can be checked by the handler to confirm whether there is work to be done. This way you can use the work handler as the standard cleanup path: rather than having to deal with cancellation and cleanup at points where items are submitted, you may be able to have everything done in the work handler itself.


For us, that means a shared variable mvstate.poweron, which we can check and return in state_handler. If it's only modified in handlers called in the system workqueue, it doesn't even need to be protected with a mutex or other synchronization primitive. You can see the actual code on [github](https://github.com/KalynaEnergy/mv).

I've also added in the [ADC]({% link _posts/2024-01-25-ac-signal-analysis.md %}) code. This is running in the main thread, which is preemptible and lower priority than the system workqueue. So far it all seems to work together.