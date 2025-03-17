# Multiplayer Snake Game with Audio and Cloud Integration: “Snake VS Snake”

By: Alex Velazquez and Hamim Sadikeen

## Description

This project implements a multiplayer version of the classic Snake game using two CC3200 microcontrollers, an OLED display, two joysticks, an ADC, an 8-ohm speaker, and AWS cloud integration. One CC3200 serves as the primary game controller, rendering the game on the OLED display and handling player movements via joysticks. The secondary CC3200 acts as a dedicated sound processor, receiving GPIO signals from the main CC3200 to generate real-time sound effects using PWM signals to drive an external speaker. Additionally, after each game session, the primary CC3200 sends a POST request to AWS, logging the game results, tracking player scores, and sending email notifications to the players.


## **Design**
### Functional Specification
![Functional Specification](https://github.com/user-attachments/assets/52846aba-d1be-49f8-88f7-4b2293053f0f)
1. When initially powered on, a start menu graphic will be presented on the screen. The program will remain in the start stage unless the start signal is given.
2. Once the start signal is given, the screen is cleared and the game is initialized from set starting conditions. The main game loop then begins. 
3. Joystick positions are decoded and used as inputs for the game logic.
4. Using the current state of the game and user inputs, the CC3200 calculates whether or not a win condition is met (states 7 and 5)
5. If the win condition is not met, then the current frame of the game is rendered to the OLED display
6. Values are passed to the DAC to play sound effects based on the current state of the game. Loops back to state 3 to get new user inputs. 
7. If the win condition is met, the game enters the finish state and an end screen is shown displaying the winning player. The program will remain in this state until the restart signal is passed. If the signal is given, then the state machine moves to state 2 to reinitialize the game.


### System Architecture
![System Architecture](https://github.com/user-attachments/assets/000ad511-ed17-49cc-b9ae-ad498d1374ed)

1. Game Logic, Display, AWS communication (Primary CC3200)
	1. Runs the Snake game on a 128x128 OLED display with SPI commands.
	2. Uses two joysticks (one per player) to control the snakes in multiplayer mode.
	3. Joysticks produce analog output which the ADS1115 acts as a 4-channel ADC and communicates with the microcontroller via I2C.
	4. Handles game logic, including collision detection, scoring, and multiplayer interactions. 
	5. Sends RESTful api commands to the AWS server via WIFI.
	6. Communicates with the secondary CC3200 via GPIO signals to trigger sound effects.
2. Audio Processing (Secondary CC3200)
   	1. Receives GPIO signals from the main CC3200 to determine which sound to play.
   	2. Generates sound effects (e.g., eating an apple, game over/victory, movement) using PWM signals to modulate an 8-ohm speaker.



## **Implementation**

### Joysticks/ADS1115

The specific joysticks can be found in our project BOM. There are 6 pins on the board that we used: SDA, SCL, X-axis, Y-axis, VCC, and GND. Both X and Y axis produce variable voltages depending on the position of the respective axis. VCC and GND pins should be connected to the board 3.3V and ground respectively. The ADS1115 requires two configuration writes with I2C from the board. These register specifications can be found in the datasheet for the ADS1115. The commands that we used were "writereg 0x48 0x01 2 0xC2 0x83", and "writereg 0x48 0x00 1 0x00". The first is a basic initialization to continuous read mode, the second points the output register to the 1st output port of the ADC (A0). Keep in mind that to read from the other ports (A1-A3), you will need to write to the ADC to read from each port every time. We accomplished this by having a function that was dedicated to reading each channel individually and returning voltages. From these voltages we used threshold logic to detect whether the joystick is up/down or left/right. An important note was that there needs to be a small delay to read all the data from the ADC in each function, a quick read and process of voltages will lead to misreading of the registers.

### Game Calculations and Display
#### int snakeGame()

- Note: Each snake has its own copy of the variables described below. They function identically, but snake 2’s functions are called below those of snake 1.
At the heart of the game is a 32x32 array of “Node” structs. Each Node has 4 fields: TTL (time to live), xPos (the x coordinate where the Node is mapped on the screen), yPos (the y coordinate where the Node is mapped on the screen), and type (determines whether the Node represents empty space, player 1, player 2, or food). The main game logic is derived from traversing and operating on this 2d Node array. From the joystick inputs, x and y-axis index modifiers are set. These modifiers determine which element of the array will be accessed next loop iteration, or tick. For example, if the joystick were pressed up, then the x-axis modifier (xArr) will be set to 0 and the y-axis modifier (yArr) will be set to -1. This corresponds to moving one row up in the array. The program then checks the “type” field of the current Node. If it is 1 or 2, then a winner is selected depending on which snake accessed the Node and what the value of “type” is, the GPIO pin for the winning sound is set HIGH, and the game loop ends. If it is 3, then the snake which accessed the food will have its “length” and “point” values increment, and a “foodCount” variable will decrement. If the “type” is 0, then the program sets the current Node as a snake node. The TTL field is set to the value of length. xPos is set to the current i index of the array times 4 (to map to the screen). Similarly, yPos is set to the current j index times 4. The type field is set to either 1 or 2 depending on the player. Depending on the values of xArr and yArr, the GPIO pins for movement sounds are either set HIGH or LOW.

Once the Node is updated, based on the type of the Node, a green or red 3x3 pixel rectangle is drawn at the (xPos, yPos) coordinate using the Adafruit_GFX() library. The xPos and yPos values will always be multiples of 4, and the slightly undersized rectangles allow for visual separation between the Nodes. This allows for the 2d array to be neatly mapped to the display. 

After this, the updateNodes() and generateFood() functions are called. At the bottom of the game loop, the current points for each player are formatted and printed at the top corners of the display. The game loop then repeats. 

#### void updateNodes(struct Node arr[33][33])

Nested for-loops are used to iterate across every element in arr. For every index, the function checks whether type == 1 or 2 and TTL == 0, indicating that the tail of the snake has moved past the Node. The values match, then the Node’s xPos and yPos values are used to draw a 3x3 black rectangle to the display, and the type field is set to 0. Otherwise, the TTL of the Node is decremented. 
This function is responsible for the snake updating as it moves forward.

#### void generateFood(struct Node arr[33][33], int foodCount, int rand1, int rand2)

The rand1 and rand2 integers are used as seeds for rand() functions. The seeds for the functions being variable helps to increase randomness. Random i and j indices (randI and randJ) are selected using these rand() functions. To determine if food needs to be generated, the function checks if the foodCount is less than 1. If it is, then a random index is selected using randI and randJ (arr[randI][randJ]) and its type is checked. If type==0, then the randomly selected index is empty and valid to become a food Node. The type of the selected Node is set to 3, the xPos and yPos fields are set to randI*4 and randJ*4, a yellow 3x3 rectangle is drawn at (xPos, yPos), and the foodCount increments. 
If the randomly-selected Node’s type is not zero, then it is already occupied and not valid to become a food Node. In this case, the program will wait for the next game tick to retry food generation.

#### Drawing Bitmap Images:

We drew simple 128x128 pixel graphics using Microsoft Paint and GIMP. Once we were satisfied with our art, we used the image2cpp tool to convert png files to bitmaps. We were able to display these bitmaps onto the display using the drawBitmap() function from the AdaFruit_GFX library.
By changing the “color” parameter in the function, we were able to draw the bitmaps in 1 solid color, with a transparent background. Because we wanted the title and end screens to have multiple colors, we had to split the images into multiple parts and layer them on top of each other using multiple drawBitmap() calls. We kept each layer 128x128 pixels for simplicity’s sake. The final screens as they appear on the display are as follows:
- Title Screen

- Player 1 Win End Screen

- Player 2 Win End Screen

- Tie End Screen






### AWS Cloud

Our incorporation of AWS into the project was identical to what we did for Lab 4. Once a game finished and the end screen was displayed. an email would be sent to each player about who won and for how many times they won. The only thing we updated was our “Thing” policy to be able to send emails to both players and a change on how the text is combined in JSON format.

### PWM/Speaker

The secondary CC3200 had code that used Systick for timing of the sound generation. We used the PWM example code from the driver library and used the TimerLoadSet() function to control the frequency of each tone. We used a RC low-pass filter at the output of the PWM with R=1k and C=1 nF to get rid of any high frequency noise. 

We then connected the output of the filter to an external speaker to be able to play sound. The sound effects were explicit functions that told the PWM to play a certain frequency for a specific amount of time. We also included an argument for amplitude modulation to be able to control the noise level for some tones to make the tone sound dynamic. We then continuously poll GPIO pins for high signals from the primary CC3200 and play functions according to the GPIO pin being pulled high. We initially planned for more advanced sound generation which would introduce more delay into the running of the base game. This would call for the purpose of the second CC3200 but because we are only implementing a PWM signal with short bursts of delay we believe that the on-screen lag will be negligent.


## Challenges

  We originally planned to have more custom sound effects using a resistor ladder DAC to produce pure sine waves. We would connect the output through an Op-Amp buffer to drive an external speaker. However we did run into limitations with our board and specifically the GPIO switching speed. The DAC utilized for audio applications requires the speaker to play frequencies in the range from 20Hz - 20kHz. Nyquist's Theorem states that to accurately reproduce an audio signal, the sample rate must be at least twice the highest frequency in the signal. For example, if the maximum audible frequency is 20 kHz, the minimum sample rate would be 40 kHz. And to get GPIO switching speed, you would have to multiply the bits and the sampling rate. We attempted to use 8 bits in our DAC which would require a switching speed of 320 kHz for our GPIO pins. Though typically for most microcontrollers the GPIO pins can switch up to the MHz range, we were not able to get pins to switch faster than about 50 Hz. This would entirely prevent our audio player from working. We are still not entirely sure if this result was a hardware problem or a software problem. Further testing will be needed to investigate the true switching speed of GPIO pins which then we could reevaluate the potential of using this specific kind of DAC. With our strict deadline we decided to use the onboard PWM signals to successfully produce square waves at various frequencies to simulate arcade sounds.
	Another challenge we met was attempting to use UART communication between boards for playing sound effects. We know that this method of communication is possible with our CC3200 boards as we have done something similar in an earlier but we ran into a problem of our UART pins not being able to produce any output. After checking the common fixes for UART such as shared ground planes and correct sysconfig we still could not get any output. We suspect that the specific pins that we used in the sysconfig cannot be actually used for UART communication. And because we were already using many pins for various other functions such as SPI, I2C and GPIO, it would have been too time consuming to switch pin assignments to different things and solve them by trial and error. Since our sound effects were not entirely sophisticated, all we needed was some signal to tell the other microcontroller to play a specific sound. We accomplished this by testing other GPIOs then using just four different pins to each control a different sound effect.


## Future Work

To make the game more engaging and fun, we would consider adding more games, high-resolution sound effects, and leaderboards. Implementing a more advanced graphics library for the OLED display could allow for smoother animations and a better visual experience. Additionally, refining the joystick controls with adaptive sensitivity based on player input could improve responsiveness and gameplay feel. To make it more cost-effective, we could shrink the entirety of the game onto a single CC3200 given more development time, reducing hardware complexity and power consumption. Furthermore, optimizing the communication protocol between the microcontrollers, such as using UART or SPI instead of GPIO signaling, could enhance performance and minimize latency in multiplayer interactions. Finally, integrating cloud-based storage for persistent leaderboards and player stats would add long-term engagement and allow for remote score tracking.


## Bill of Materials
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
