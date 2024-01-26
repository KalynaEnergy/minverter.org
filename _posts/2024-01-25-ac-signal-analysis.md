---
layout: post
title: AC signal analysis
tags: software
categories: 
usemathjax: true
---

With the ability to [take data]({% link _posts/2023-08-21-adc.md %}) in hand, we need the capability of analyzing it, and determine both AC and DC characteristics of the signals. 
<!-- more -->
First, let's discuss the circuit and hardware configuration. The [front end voltage divider]({% link _posts/2023-08-22-adc-frontend.md %}) gives us a gain of 2*6.8k/820k and shifts the signal to the middle of the range, giving us a nominal input range of +-398 V based on 3.3 V VDD. We have a big input impedance, so we need a long averaging time. The [maximum TACQ selectable](https://infocenter.nordicsemi.com/index.jsp?topic=%2Fps_nrf52840%2Fsaadc.html) is 40 us, and is suitable for a 800 kOhm source impedance. We will still measure VDD, so we can shift appropriately, but it won't need a long sample aquisition time. We can do some averaging in hardware by setting oversampling, to reduce random noise, but we will do that manually in software as part of the analysis. Here's the overlay:

```
&adc {
	#address-cells = <1>;
	#size-cells = <0>;

	channel@0 {
		reg = <0>;
		zephyr,gain = "ADC_GAIN_1_6";
		zephyr,reference = "ADC_REF_INTERNAL";
		zephyr,acquisition-time = <ADC_ACQ_TIME(ADC_ACQ_TIME_MICROSECONDS, 40)>; // was ADC_ACQ_TIME_DEFAULT
		zephyr,input-positive = <NRF_SAADC_AIN1>; /* P0.03 */
		zephyr,resolution = <12>;
		zephyr,oversampling = <0>;
	};

	channel@7 {
		reg = <7>;
		zephyr,gain = "ADC_GAIN_1_6";
		zephyr,reference = "ADC_REF_INTERNAL";
		zephyr,acquisition-time = <ADC_ACQ_TIME(ADC_ACQ_TIME_MICROSECONDS, 3)>;
		zephyr,input-positive = <NRF_SAADC_VDD>;
		zephyr,resolution = <12>;
		zephyr,oversampling = <0>;

	};
};
```
The input is connected to P0.03, and a cap to ground can be connected also to give a RC filter with a corner of 1/2pi 6.8k C. 

Next, let's discuss block data capture. It's inefficient to have the processor collecting each data point by hand, waiting, and we'd like to get reasonable data rates out. We first set up a buffer for the data to be collected into:

```
#define BLOCK_SIZE 4096
uint16_t raw_data[BLOCK_SIZE*2] = {0};

```

and an adc_sequence structure connected to this buffer. We also add an adc_sequence_options structure which we will need
```
	struct adc_sequence sequence = {
		.buffer = &raw_data[0],
		/* buffer size in bytes, not number of samples */
		.buffer_size = sizeof(raw_data),
	};
	struct adc_sequence_options opts = {
		.extra_samplings = BLOCK_SIZE-1U,
	};
```

Now we configure the sequence, first with the adc_sequence_init_dt API call, then manually modifying it to sample from all desired channels, with the sequence options to tell it how many times to run through the sequence. Then adc_read() will bring an entire block of input data and VDD  
```
		// first, configure sequence using channel 0. channel number doesn't matter
		// sequence.channels will be incorrect, we will fix after
		(void)adc_sequence_init_dt(&adc_channels[0], &sequence);
		sequence.channels = BIT(0) | BIT(7); // we want both channel 0 and 7
		sequence.options = &opts; // to get the extra_samplings to get the entire block
		err = adc_read(adc_channels[0].dev, &sequence); // I think what device is linked doesn't depend on the channel
		if (err < 0) {
			printk("Could not read (%d)\n", err);
			continue;
		} 
```

Next we can scale the data in millivolts with the API, as in the example, followed by offsetting and scaling to physical volts
```
#define VOLTAGE_DIVIDER_SF 0.241f // scale factor 241 is 2*820k/6.8k, *1e-3 (mV to V)
float32_t sample_data[BLOCK_SIZE] = {0.f};

...

		// let api scale to mV using devicetree
		int32_t v0_mv, vdd_mv;
		for (size_t i = 0; i < BLOCK_SIZE; i++) {
			v0_mv = raw_data[2*i+0]; 
			err = adc_raw_to_millivolts_dt(&adc_channels[0],
									&v0_mv);
			if (err < 0) {
					printk(" (value in mV not available)\n");
			}
			vdd_mv = raw_data[2*i+1]; 
			err = adc_raw_to_millivolts_dt(&adc_channels[1],
						&vdd_mv);
			if (err < 0) {
					printk(" (value in mV not available)\n");
			}
			// then offset and scale to volts based on voltage dividers
			sample_data[i] = VOLTAGE_DIVIDER_SF*(v0_mv - vdd_mv/2); 
		}


```
We can just average these results to measure DC voltages, and they aren't too terrible. The offset is off by a couple of volts, the scale factor by percents, but I built the voltage divider with 5% resistors. Precision voltage dividers for ADCs is why 0.1% resistors exist. A proper circuit would use them. Alternatively I could measure and correct the values here, but this is close enough for the moment.

Now let's do some AC analysis. IEEE 1547 [says]({% link _posts/2023-08-21-adc.md %}) we need to measure RMS voltage to 1% in 10 cycles, frequency to 10 mHz in 60 cycles. With a full time series in hand, we can do an [FFT](https://en.wikipedia.org/wiki/Fast_Fourier_transform) and analyze the signal in frequency domain. This is not the only way to do it, perhaps not even the best way to do it, but it is a way. Writing FFT code seems like fun, but I'm lazy. Fortunately there's a package maintained by AMD included in NRF Connect SDK: [CMSIS DSP](https://arm-software.github.io/CMSIS_5/DSP/html/index.html). Hopefully it can work okay. We need a few things to get going. First, add to project.conf:

```
CONFIG_FPU=y
CONFIG_CMSIS_DSP=y
CONFIG_CMSIS_DSP_TRANSFORM=y
CONFIG_CMSIS_DSP_COMPLEXMATH=y
CONFIG_CMSIS_DSP_STATISTICS=y
```
And then places to stash data:

```

#define SAMPLE_RATE 16340.4f // constant is sampling rate, determined by experimental calibration

float32_t data_detrend[BLOCK_SIZE] = {0.f};
float32_t fftout[BLOCK_SIZE/2+2]; // output of real FFT
float32_t ps[BLOCK_SIZE]; // power spectrum, in V^2 for each bin (*not* distribution in V^2/Hz)
float32_t block_window[BLOCK_SIZE]; // window, needs to be computed only once
float32_t window_sum, window_sumsq; // window normalizations
static arm_rfft_fast_instance_f32 arm_rfft_S; // needs to be computed only once

```
We might have expected a sample rate of 1/(40 us + 3 us + 2*2 us), using the aquisition times and conversion times for each channel. This works out to about 21 ksps. The sample rate I put in there came from analysis of a signal from a signal generator, which should be pretty accurate. It differs from the calculated value by 20%, so there are other delays not included, or the timing on the chip isn't perfect. I don't even know if there's a crystal, so perhaps the effective clock rate is different than what the calculation is expecting. It doesn't seem critical however.

Before analyzing a signal with FFT, it normally makes sense to apply a [window](https://en.wikipedia.org/wiki/Window_function). There have been thousands of pages of ink spilled on the subject. CMSIS cites [this](https://pure.mpg.de/rest/items/item_152164/component/file_152163/content) report, which is a pretty good summary of general issues in signal processing, plus has a few new window functions. Mostly it doesn't matter, as long as you pick something, and understand some of the tradeoffs. I'm trying to measure amplitudes of AC signals to 1%, but the signal frequency might be anywhere inside the closest frequency bin. So, I need a flat top function. I picked the HFT95 function, which is good to about 0.05%. While I'm at it I can sum the power in harmonics to compute [THD](https://en.wikipedia.org/wiki/Total_harmonic_distortion), for which IEEE 1547 says I can have up to 5%, and noise. For the noise I skip bins close to the dominant tone and harmonics, and otherwise mix everything together. Note that noise is broad spectrum, while tones are narrow, so they have different units (volts vs volts/rtHz) even if they sometimes appear in the same plot. Yes, it can be confusing; no it mostly doesn't matter. I also remove the DC before the FFT, but don't try to remove any linear trend. My assumption is that, at least most of the time, I'll be working with sources without slow changes in the DC level. The initialization code looks something like this:


```

	// fft initialization

	status = arm_rfft_fast_init_f32(&arm_rfft_S, BLOCK_SIZE);
	if (status != ARM_MATH_SUCCESS) {
		printk("arm_rfft_fast_init failure\n");
	}
	arm_hft95_f32(block_window, BLOCK_SIZE); // window function, good to about 0.05% amplitude, ~4 bins wide
//	arm_accumulate_f32(block_window, BLOCK_SIZE, &window_sum);
	window_sum = window_sumsq = 0.f;
	for (size_t i=0; i< BLOCK_SIZE; i++) {
		window_sum += block_window[i];
		window_sumsq += SQR(block_window[i]);
	}
```
and the calculation for each block is like this

```
		// now stats and fourier w dsp library
		status = ARM_MATH_SUCCESS;
		float32_t maxValue, meanValue;
		uint32_t maxIndex;

		float32_t binWidth = (SAMPLE_RATE/BLOCK_SIZE); 
		arm_mean_f32(sample_data, BLOCK_SIZE, &meanValue);
		arm_offset_f32(sample_data, -meanValue, data_detrend, BLOCK_SIZE);
		arm_mult_f32(data_detrend, block_window, data_detrend, BLOCK_SIZE);
		arm_rfft_fast_f32(&arm_rfft_S, data_detrend, fftout, 0);
		// for (size_t i = 0; i < 35; i++) {
		// 	printk("%5.2f\t%10.2f\t%10.2f\n", binWidth*i, fftout[2*i], fftout[2*i+1]);
		// }
		ps[0] = 0.f; // zero out DC from power spectrum (also f_nyquist is in here too)
		arm_cmplx_mag_squared_f32(&fftout[1], &ps[1], BLOCK_SIZE/2-1);
		arm_max_f32(ps, BLOCK_SIZE, &maxValue, &maxIndex);
		if (maxIndex < 5) {
			printk("Max power index %" PRId32 " too small, results will be wrong\n", maxIndex);
		}
		float32_t tonePower = ps[maxIndex]; 
		float32_t harmonicPower = 0.f;
		size_t k = 0U; // will end at number harmonics plus one
		for (k = 2; k < 51; k++ ) { // 50 harmonics or whatever is inside our bandwidth
			if (k*maxIndex >= BLOCK_SIZE/2) {
				break;
			}
			harmonicPower += ps[k*maxIndex];
		}

		float64_t noisePower = 0.f;
		int32_t noiseBins = 0U;
		for (size_t i = 0; i < BLOCK_SIZE/2; i++) {
			size_t ii = i % maxIndex;
			if (ii == maxIndex-2 || ii == maxIndex - 1 || ii == 0 || ii == 1 || ii == 2) {
				; // skip bins counted for tone or harmonics, or a couple bins either side
			} else {
				noisePower += ps[i];
				noiseBins++;
			}
		}
		harmonicPower -= (k-2)*noisePower/noiseBins; // subtract white noise background from harmonic distortion measurement
		if (harmonicPower < 0.f) {
			harmonicPower = 0.f; // if harmonic distortion outweighed by noise, display 0.
		}
		printk("DC %.2f Tone: %.2f Hz mag %.2f Vrms phase %.3f rad THD %.2f%% rms noise %.2f V/rtHz\n", 
			meanValue,
			binWidth*maxIndex,
			sqrt(2*tonePower/SQR(window_sum)), 
			atan2(fftout[2*maxIndex+1], fftout[2*maxIndex]), 
			100.f*sqrt(harmonicPower/tonePower),
			sqrt(2.f*noisePower/(binWidth*noiseBins*window_sumsq))
			);
```
Overall, I get amplitude values that are about right, within a percent or so of the expected value, not popping around too much. Noise is 0.5-1.5 V/rtHz, increasing a bit when I turn the voltage up on the signal generator. These blocks are about a quarter of a second. Not bad for signal levels more than an order of magnitude smaller than the design input range.

Okay, so that's voltage. What about frequency? Well, you can't pick the maximum bin with any greater precision than the width of the bins themselves. And the bin width has nothing to do with sampling rate or resolution or accuracy, it's 1/T where T is the total sampling time. If 60 cycles = 1 second, this means we have a frequency accuracy of 1 Hz. This is not driven by our hardware or algorithms, it's not even a law of nature (though it's related to Heisenberg's uncertainty principle), it's essentially a law of mathematics. A chunk of a wave 1 second long *doesn't have* a frequency to anything better than 1 Hz. It's a basic characteristic of windowing data. The spec, though, says we need to measure this to 0.01 Hz! Are the spec writers stupid? Almost certainly not. Rather, we need to interpret the spec not in a general mathematical context, but rather specifically in terms of what actual signals the system will be exposed to, and what the testing procedure for spec compliance will be. To use other language, we have a strong prior about the signal, which we are allowed to use. We need to update a frequency result every 60 cycles, and (under test conditions) it must match the input frequency to 0.01 Hz, but we are not required to break any laws of physics or mathematics. The code above gives the input frequency correct to the closest bin, which is a few Hz. To do better, we have to somehow average over longer periods, and use our knowledge of the input. 