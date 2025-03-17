# Multiplayer Snake Game with Audio and Cloud Integration: “Snake VS Snake”

By: Alex Velazquez and Hamim Sadikeen

## 1) Description

This project implements a multiplayer version of the classic Snake game using two CC3200 microcontrollers, an OLED display, two joysticks, an ADC, an 8-ohm speaker, and AWS cloud integration. One CC3200 serves as the primary game controller, rendering the game on the OLED display and handling player movements via joysticks. The secondary CC3200 acts as a dedicated sound processor, receiving GPIO signals from the main CC3200 to generate real-time sound effects using PWM signals to drive an external speaker. Additionally, after each game session, the primary CC3200 sends a POST request to AWS, logging the game results, tracking player scores, and sending email notifications to the players.


## 2) Design

Design

## 3) Implementation

Implementation

## 4) Challenges

  We originally planned to have more custom sound effects using a resistor ladder DAC to produce pure sine waves. We would connect the output through an Op-Amp buffer to drive an external speaker. However we did run into limitations with our board and specifically the GPIO switching speed. The DAC utilized for audio applications requires the speaker to play frequencies in the range from 20Hz - 20kHz. Nyquist's Theorem states that to accurately reproduce an audio signal, the sample rate must be at least twice the highest frequency in the signal. For example, if the maximum audible frequency is 20 kHz, the minimum sample rate would be 40 kHz. And to get GPIO switching speed, you would have to multiply the bits and the sampling rate. We attempted to use 8 bits in our DAC which would require a switching speed of 320 kHz for our GPIO pins. Though typically for most microcontrollers the GPIO pins can switch up to the MHz range, we were not able to get pins to switch faster than about 50 Hz. This would entirely prevent our audio player from working. We are still not entirely sure if this result was a hardware problem or a software problem. Further testing will be needed to investigate the true switching speed of GPIO pins which then we could reevaluate the potential of using this specific kind of DAC. With our strict deadline we decided to use the onboard PWM signals to successfully produce square waves at various frequencies to simulate arcade sounds.
	Another challenge we met was attempting to use UART communication between boards for playing sound effects. We know that this method of communication is possible with our CC3200 boards as we have done something similar in an earlier but we ran into a problem of our UART pins not being able to produce any output. After checking the common fixes for UART such as shared ground planes and correct sysconfig we still could not get any output. We suspect that the specific pins that we used in the sysconfig cannot be actually used for UART communication. And because we were already using many pins for various other functions such as SPI, I2C and GPIO, it would have been too time consuming to switch pin assignments to different things and solve them by trial and error. Since our sound effects were not entirely sophisticated, all we needed was some signal to tell the other microcontroller to play a specific sound. We accomplished this by testing other GPIOs then using just four different pins to each control a different sound effect.


## 5) Future Work

To make the game more engaging and fun, we would consider adding more games, high-resolution sound effects, and leaderboards. Implementing a more advanced graphics library for the OLED display could allow for smoother animations and a better visual experience. Additionally, refining the joystick controls with adaptive sensitivity based on player input could improve responsiveness and gameplay feel. To make it more cost-effective, we could shrink the entirety of the game onto a single CC3200 given more development time, reducing hardware complexity and power consumption. Furthermore, optimizing the communication protocol between the microcontrollers, such as using UART or SPI instead of GPIO signaling, could enhance performance and minimize latency in multiplayer interactions. Finally, integrating cloud-based storage for persistent leaderboards and player stats would add long-term engagement and allow for remote score tracking.


## 6) Bill of Materials
| Component       | Quantity | Price  | Obtainment               |
|----------------|----------|--------|--------------------------|
| [CC3200](https://www.digikey.com/en/products/detail/texas-instruments/CC3200-LAUNCHXL/4862812?&utm_adgroup=Texas%20Instruments&utm_term=&gad_source=1)    | 2        | $66.00 | From lab supplies        |
| [Joysticks](https://www.amazon.com/dp/B0DSZ9D9WH?psc=1&smid=ABNC1ZV0O9ATN&ref_=chk_typ_imgToDp) | 2        | $8.80  | Amazon.com               |
| [ADS1115](https://www.amazon.com/HiLetgo-Converter-Programmable-Amplifier-Development/dp/B01DLHKMO2?source=ps-sl-shoppingads-lpcontext&ref_=fplfs&psc=1&smid=A30QSGOJR8LMXA&gQT=2)   | 1        | $7.99  | Amazon                   |
| [SSD1351 OLED](https://www.amazon.com/1-5inch-RGB-OLED-Module-Communicating/dp/B07DB5YFGW?source=ps-sl-shoppingads-lpcontext&ref_=fplfs&psc=1&smid=A50C560NZEBBE&gQT=2) | 1     | $23.99 | From lab supplies        |
| [8 Ohm Speaker](https://www.amazon.com/Speaker-Speakers-Compatible-Loudspeaker-Player/dp/B09YGDQV3Z/ref=sr_1_3?dib=eyJ2IjoiMSJ9.5-gTee9dr0E3dA0xSDxZywREWn7rpS35zhwyI-pUOlgZvRYjW-joHN_NVQbgBKJAX93NCu61XhVDa_VAM9qkOm-R_iAALgfCViuFBtEWT4C3TGOeakzqbszQVgs2C6M2ikPyqMxdGn4s4rFyxHF_CcQKNS8sXgLgXC0SmqEAw1GjazqhUw07BYdrdeP-VtEH30DmQugPZW-t92XnXx6udKd__xRIA6Gg3nnkuYoODuiAYRiPq54KTl6UtQWW5elJUj4omvgpzQGbjTwkx8ixoKFKf-9MXVZVA3KaxMK9jPc5kG2YxZdmp-Q_5QRwzu7Y24Eqwc_Bd8N0LvjLANOTzwlXNQq981R_bcoi9ch6KwQ.UaqwL4ZPXmICLWA4M9IUno51-cJJ5L5XLeIXaNmINgM&dib_tag=se&keywords=8%2Bohm%2Bsmall%2Bspeaker&qid=1741067219&s=electronics&sr=1-3&th=1) | 1    | $1.00  | From lab supplies        |
| [Jumper Wires](https://www.amazon.com/Elegoo-EL-CP-004-Multicolored-Breadboard-arduino/dp/B01EV70C78/ref=sr_1_3?s=electronics&sr=1-3) | 1     | $6.98  | From lab supplies        |
| **Total:**     |          | **$180.76** |                      |
| **Total excluding lab / personal supplies:** | | **$16.79** | |
