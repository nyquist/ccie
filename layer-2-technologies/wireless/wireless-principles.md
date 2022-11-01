# Wireless Principles

## RF Spectrum

A radio wave is an electromagnetic field radiating from a sender. The wave propagates towards a receiver which receives its energy. Electromagnetic waves are caracterized by their wave length. The length of the wave is defined as the physical distance the wave covers in one cycle so it is $$\lambda ={\frac {v}{f}}\,\,,$$wher v is the speed of the wave and f is the frequency of the wave. Since Electromagnetic waves travel at the speed of light $$c = 3 x10^8 m/s$$, the wavelength is $$\lambda ={\frac {c}{f}}\,$$  . For example an electromagnetic (radio) wave of 100 MHz has a wave length of&#x20;

$$
\lambda ={\frac {3\times 10^8 m/s}{100 \times 10^6 Hz} } =  {\frac {3 \times 10^8 m/s}{1 \times 10^8 1/s}} = 3m
$$

since $$1 Hz = \frac{1}{s}$$ and represents the frequency of the wave.

A wave also has an amplitude which represents the ammount of energy that is injected in one cycle. The amplitude of a wave can be increased through amplification. Amplification can be active (more energy is applied) or passive (energy is focused with an antenna). Decreasing the amplitude is called attenuation. While travelling further from the source wave amplitude suffers attenuation.As energy is absorbed by other obstacles on the path, attenuation can happen much faster depending on the environment. But there are also reasons for weaker signals at receivers which make up the "Free Path Loss"

* The signal is snet in all directions so the sender's energy is spread in all directions. With antennas energy can be focused in certain areas but there is no perfect way of focusing the energy from sender to receiver
* The receiver has a certain size and can only collect a limited ammount of the energy that is sent.

Regulations are in place to determine the (maximum) ammout of power that should be used for each device type based on the distance where the signal is expected to be sent.

## RSSI and SNR

To determine how much of the original signal reaches the reciever we can use RSSI (Received Signal Strenght Indicator) and SNR (Signat to Noise Ratio). RSSI - Noise = SNR

RSSI calcualtion is not easy because the reciever doesn't know how much power was originally sent so RSSI is a relative value obtained by comparing received packets to each other. Each vendor has it's own ranges so the same RSSI value from one vendor can't be compared with the RSSI value from another vendor. For Cisco, a good RSSI value is higher than -67dBm.

RCPI (Received Channle Power Indicator) is an attempt to standardize RSSI across vendors.

The noise represents the interferences in your noise level so lower is better (e.g -95dBm).&#x20;

SNR represents the ratio between Signal and Noise but can be obtained with the formula SNR = RSSI - Noise. Since RSSI and Noise are negative values in dBm the result should end up positive in most situations. A SNR>20dBm is good.

SINR (Signla to Interference plus Noise Ratio) = RSSI - (interference + noise). A SINR > 25dBm is required for voice over wireless

## Decibels and Watts

Power is measured in Watts but power calculations can be a complex excercise. For this reason, a simpler apporach is to use the logarithmic measurement that expresses the ammount of power relative to a reference, which is called decibel (dB)

When the reference power is equal to the compared power, the dB difference between them is 0. In addition, these shortcuts can be used to quickly evaluate the power value based on it's reference

* \+10 dB means the compared value is 10 times more powerful than the reference.
* \+ 3 dB means the compared value is 2 times more powerful than the reference
* \-3 dB means the compared value is half the power of the reference
* \-10 dB means the compared value is 1/10 of the power of the reference.

Sometimes the decibel value also has a reference indicator:

* dBm - the reference value is a power of 1mW
* dBd - the reference value is a [dipole antenna](https://en.wikipedia.org/wiki/Dipole\_antenna)&#x20;
* dBi - the reference value is an isotropic antenna (an omnidirectional antenna)

EIRP (Effective Isotropic-Radiated Power) \[dBm] = TX \[dBm] + Antenna\[dBi] + Cable Loss\[dB]

## Antenna Characteristics

Antennas fall in 2 main categories:

* Omnidirectional: They radiate a signal with the same strength in all directions so the energy is evenly distributed
  * Dipole
* Directional: the directional antenna radiates most of the energy in some direction so it is not evenly distributed. Because they are stronger in some areas they add "gain".
  * Yagi
  * Patch

The term directional here refers to all directions in a 3D space but vendors typically provide 2x 2D representations for the Azimuth (horizontal) and the elevation plane.

