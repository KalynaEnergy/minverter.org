
---
layout: post
title: Changing PWM pulse lengths
tags: hardware, firmware
categories: 
---

So what caused the [glitch]({% link _posts/2024-01-30-glitch.md %}) we saw? One clue was that it only happened while the duty cycle was being lowered, not while it was increasing. So clearly something messed about how the pulse widths were being changed. If you think about centered pulses, opposite polarities, and trying to change the overall width to change the duty cycle, there's a problem. You have to change one first, then the other. It's not atomic (at least as I've implemented it). It doesn't take that long, but the possibility is that one changes before the other. That's okay, if the one that changes first effectively increases the dead time for a cycle. If you get the order wrong, the dead time is *decreased*, and if the width change is large enough, goes negative leading to shoot through. That's what I was seeing. The right thing to do is to choose which side to change first depending on which way the change is going. Here is working code:

overlay:

```
/{
    custompwms {
        compatible = "pwm-leds";
    
        custompwm0: custom_pwm_0 {
            pwms =  <&pwm0 0 PWM_MSEC(20) PWM_POLARITY_INVERTED>,  // period and flag values will be overwritten
                    <&pwm0 1 PWM_MSEC(20) PWM_POLARITY_NORMAL>,
                    <&pwm0 2 PWM_MSEC(20) PWM_POLARITY_NORMAL> 
                   ;
        };
    };
    aliases {
        mycustompwm = &custompwm0;
    };
};

&pwm0 {
    center-aligned;
};

&pinctrl {
    pwm0_default: pwm0_default {
		group1 {
			psels = <NRF_PSEL(PWM_OUT0, 0, 24)>,    // P0.24, red LED
				    <NRF_PSEL(PWM_OUT1, 0, 27)>,    // P0.27 = Arduino Nano D9, LS switch
                    <NRF_PSEL(PWM_OUT2, 0, 21)>;    // P0.21, Arduino Nano D8, HS switch
		};
	};
};
```

main.C:
```
#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>
#include <zephyr/device.h>
#include <zephyr/drivers/pwm.h>
#include <math.h>

static const struct pwm_dt_spec custompwm0 = PWM_DT_SPEC_GET(DT_ALIAS(mycustompwm));

#define PWM_FREQ 1000 // Hz
#define CYCLE_FREQ 0.3 // Hz

#define STEPS 8
float levels[STEPS]; // holds duty cycles, duty = what fraction of time HS switch is on
#define DUTY_AVG 0.1f
#define DUTY_RANGE 0.05f
#define DEADTIME_NS 500U

#define TWO_PI 6.28318530718f

int main(void)
{
	uint32_t ret;

	if (!device_is_ready(custompwm0.dev)) {
		printk("Error: PWM device %s is not ready\n",
		       custompwm0.dev->name);
		return 0;
	}

	for (int i =0; i < STEPS; i++) {
		levels[i] = DUTY_AVG*(1 + DUTY_RANGE*sin(TWO_PI*i/STEPS));
	}

	uint32_t count = 0;
	uint32_t oldpulsewidth_ns = 0;
	while (1) {
		uint32_t pulsewidth_ns = PWM_HZ(PWM_FREQ)*levels[count++ % STEPS]; 
		ret = pwm_set_dt(&custompwm0, 
			PWM_HZ(PWM_FREQ), 
			pulsewidth_ns);
		if (pulsewidth_ns > oldpulsewidth_ns) {
			ret |= pwm_set(custompwm0.dev, 1,
				PWM_HZ(PWM_FREQ), 
				pulsewidth_ns + DEADTIME_NS, PWM_POLARITY_INVERTED);
			ret |= pwm_set(custompwm0.dev, 2,
				PWM_HZ(PWM_FREQ), 
				pulsewidth_ns - DEADTIME_NS, PWM_POLARITY_NORMAL);
		} else {
			ret |= pwm_set(custompwm0.dev, 2,
				PWM_HZ(PWM_FREQ), 
				pulsewidth_ns - DEADTIME_NS, PWM_POLARITY_NORMAL);
			ret |= pwm_set(custompwm0.dev, 1,
				PWM_HZ(PWM_FREQ), 
				pulsewidth_ns + DEADTIME_NS, PWM_POLARITY_INVERTED);
		}
		oldpulsewidth_ns = pulsewidth_ns;
		if (ret) {
			printk("Error %d: failed to set pulse width\n", ret);
			return 0;
		}
		k_sleep(K_USEC(1000000U/(CYCLE_FREQ*STEPS)));
	}
	return 0;
}
```

No more blips. 