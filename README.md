# ECE 528/L - Robotics and Embedded Systems with Lab

**CSU Northridge**

**Department of Electrical and Computer Engineering**

## Motor Control Lab

## Overview: 
This lab introduces and implements the basics of Pulse Width Modulation (PWM), in terms of how to generate it with system timers. In doing so, we verify this PWM using an oscilloscope. These PWM signals are then used to control servos and drive motors. Additionally, the lab sets up the GPIO pins for the bumper switches to use edge-triggered interrupts and handle collision events, and see the switch bouncing on an oscilloscope using the bumper switches. These interrupts are global variables that are to be used for different interrupt tasks.

---
### Components Used: 

* TI MSP432 LaunchPad
* User LEDs of the TI MSP432 LaunchPad
* Pololu Gearmotor with Encoder - [Product Link](https://www.pololu.com/product/3675)
* Left Bumper Switches for TI-RSLK MAX - [Product Link](https://www.pololu.com/product/3673)
* Right Bumper Switches for TI-RSLK MAX - [Product Link](https://www.pololu.com/product/3674)
* HS-485HB Servo-Stock Rotation - [Product Link](https://www.servocity.com/hs-485hb-servo/)


---
### Analysis and Results:

#### Initial Code 
The initial code provided had the "Timer_A2" Driver completed, so that it "generates two PWM signals with a refresh rate of 50 Hz". Using this, we are able to initially capture some O-Scope images of the generated PWM signals as well as connect servos to these signals to see the servos toggling positions between 0 and 180 degrees. 

The only thing needed for this was to go into the `Main.c` file and ensure the while loop looked as shown below, ensuring that the correct items were uncommented and the remaining items were commented out.

***Initial Code - Main.c***
```c
    while(1)
    {
        // Rotate to 0
       Timer_A2_Update_Duty_Cycle_1(1700);
       Timer_A2_Update_Duty_Cycle_2(1700);
       LED2_Output(RGB_LED_RED);
       Clock_Delay1ms(3000);

       // Rotate to 180
       Timer_A2_Update_Duty_Cycle_1(7000);
       Timer_A2_Update_Duty_Cycle_2(7000);
       LED2_Output(RGB_LED_BLUE);
       Clock_Delay1ms(3000);

        // Drive_Pattern_1();

        // if (collision_detected == 1)
        // {
        //     Handle_Collision();
        // }
        // else
        // {
        //     Motor_Forward(4500, 4500);
        // }
    }

```

With this small change and the initial code, we can follow the tables below to capture the screenshots of the PWM signal generation and see the servos move.

***Pinouts and Wiring Tables***

![O-Scope Connection](/Screenshots/Inital_Code_O_scope-Probing.png)
![Servos Connection](/Screenshots/Inital_Code_Servo_Connection.png)


***Artifacts for PWM Control***

Following these tables, the images displayed below were collected. In order, the first image displays the PWM signal when the servos are at 0 degrees, which is equivalent to having an actual pulse width of 600us. The following image after that displays the PWM signal with a pulse width of 2.4ms, which is tied to when the servos are at 180 degrees. 

![Zero Degree](/Screenshots/ece528L_lab1_servo_0_degrees_pulse_width_group_group_17.jpeg)
![180 Degree](/Screenshots/ece528L_lab1_servo_180_degrees_pulse_width_group_group_17.jpeg)

Images are slightly out of time, but we can see both images display the values shown in the table below. 


|Servo Angle| Pulse Width|
|-----------|------------| 
| 0 Degrees | 600us      |
|180 Degrees| 2.4ms      |

---
#### Modifications to Source Code (Tasks)

**Enabling Bumper Switches**

The first task was to set up the "Bumper_Switches_Init" found in `Bumper_Switches.c`, which would enable the use of the bumper switches on the development board. This was done by setting the six bumper switch pins (P4.0, P4.2, P4.3, P4.5, P4.6, and P4.7) to be set as GPIO inputs with their pull-up resistors enabled. They were then set to trigger on a falling edge, any existing interrupt flags were cleared, and their interrupts were enabled. Finally, the interrupt priority was set to 0, and the Port 4 interrupt (IRQ 38) was enabled in the NVIC. All this is shown being implemented in the following function found in `Bumper_Switches.c`.

***Bumper_Switches_Init Function***
```c
void Bumper_Switches_Init(void(*task)(uint8_t))
{
    // Store the user-defined task function for use during interrupt handling
    Bumper_Task = task;

    // Configure the following pins as GPIO pins: P4.7, P4.6, P4.5, P4.3, P4.2, and P4.0
    // by clearing the corresponding bits in the SEL0 and SEL1 registers

    P4->SEL0  &= ~0xED;  // // // clearing bits using masks 1110 1101 - DSR
    P4->SEL1  &= ~0xED;  // // // clearing bits using masks 1110 1101 - DSR


    // Set the direction of the following pins as input: P4.7, P4.6, P4.5, P4.3, P4.2, and P4.0
    // by clearing the corresponding bits in the DIR register

    P4->DIR  &= ~0xED;   // // clearing bits using masks 1110 1101 - DSR

    // Enable pull-up resistors on the following pins: P4.7, P4.6, P4.5, P4.3, P4.2, and P4.0
    // by setting the corresponding bits in the REN register

    P4->REN  |= 0xED;    // setting bits using masks 1110 1101 - DSR

    // Ensure that the pins are pulled up: P4.7, P4.6, P4.5, P4.3, P4.2, and P4.0
    // by setting the corresponding bits in the OUT register

    P4->OUT  |= 0xED;    // setting bits using masks 1110 1101 - DSR

    // Interrupt Edge Select: High-to-Low Transition
    // Configure the pins to use falling edge event triggers: P4.7, P4.6, P4.5, P4.3, P4.2, and P4.0
    // by setting the corresponding bits in the IES register

    P4->IES  |= 0xED;    // setting bits using masks 1110 1101 - DSR

    // Clear any existing interrupt flags on the following pins: P4.7, P4.6, P4.5, P4.3, P4.2, and P4.0
    // by clearing the corresponding bits in the IFG register

    P4->IFG  &= ~0xED;   // // clearing bits using masks 1110 1101 - DSR


    // Enable interrupts on the following pins: P4.7 - P4.5, P4.3, P4.2, and P4.0
    // by setting the corresponding bits in the IE register

    P4->IE  |= 0xED;     // setting bits using masks 1110 1101 - DSR

    // Set the priority level of the interrupts (IRQ 38) to 0 (section 2.4.3.20)
    NVIC->IP[9] = (NVIC->IP[9] & 0xFF0FFFFF);

    // Enable Interrupt 38 in NVIC (section 2.4.3.2)
    // Bit 6 corresponds to IRQ 38
    NVIC->ISER[1] = 0x00000040;
}
```
***Bumper_Switches_Handler Function***

In addition, the following was added to the `main.c` function in order to see the bumper switch working in the terminal of CCS Code Composer. 
```c
void Bumper_Switches_Handler(uint8_t bumper_switch_state)
{
printf("Collision Detected! Bumper Switch State: 0x%02X\n",bumper_switch_state);
P8->OUT |= 0x80;
}
```

In doing so, we can load and flash the code to the development board and collect the following artifacts. The first one displays serial communication from our computer to the development board and displays the bumper switches being activated when we press a switch. The following image displays the physical layout of the switches. The last image displays the O-Scope showing the physical phenomenon of switch bounce when pressed, showing the effect seen in the first image that even one press might display multiple readouts of the switch.

![Serial Readout Bumper Switches](/Screenshots/ece528L_lab1_bumper_terminal_output_group_17.png)
![Bumper Switches Layout](/Screenshots/bumper-switch-layout.png)
![Bumper Switches Bounce Effect](/Screenshots/ece528L_lab1_bumper_scope_shot_group_group_17.jpg)



**Initializing Timer_A0**

The following task was to initialize "Timer A0" using the already given `Timer_A2_PWM.c` as a reference. Below is the "Timer_A0_PWM_Init" function that was implemented in `Timer_A0_PWM.c`. 

This was done, in short, by setting Timer A0 to generate two PWM signals for the motors. The function first checked that neither duty cycle value was greater than or equal to the period and exited early if one was. P2.6 and P2.7 were then set to peripheral mode and configured as outputs. The period was loaded into CCR[0], and the EX0 register was cleared. CCTL[3] and CCTL[4] were both set to Toggle/Reset mode, with the two duty cycle values loaded into CCR[3] and CCR[4]. Finally, Timer A0 was set to use the 12 MHz SMCLK, divided by 8, running in Up/Down mode.

***Timer_A0_PWM_Init Function***
```c

void Timer_A0_PWM_Init(uint16_t period_constant, uint16_t duty_cycle_1, uint16_t duty_cycle_2)
{
    // Return immediately if either duty cycle values are greater than
    // or equal to the given period_constant
    if (duty_cycle_1 >= period_constant) return;
    if (duty_cycle_2 >= period_constant) return;

    // Configure pins P2.6 (PM_TA0.3) and P2.7 (PM_TA0.4) to use peripheral function mode
    // by setting Bits 6 and 7 of the SEL0 register for P2
    // and clearing Bits 6 and 7 of the SEL1 register for P2

    P2->SEL0 |= 0xC0 ;   // setting bits using masks 1100 0000 - DSR
    P2->SEL1 &= ~0xC0 ;  // clearing bits using masks 1100 0000 - DSR


    // Configure pins P2.6 and P2.7 as output GPIO pins to drive the PWM signals
    // Set Bits 6 and 7 of the DIR register for P2

    P2->DIR |= 0xC0 ;  // setting bits using masks 1100 0000 - DSR


    // Set the Timer A0 Capture/Compare register to the specified period_constant
    // CCR[0] is primarily used as the "period" register
    // General formula: Period = (2*period_constant) / (12 MHz / Prescale Value)
    // In this case: Period = (2*15000) / (12 MHz / 8) = 20 ms
    // Assign the value of period_constant to the CCR[0] register

    TIMER_A0->CCR[0] = period_constant ;  // Setting Timer A0  Capture/Compare register to period constant variable - DSR

    // Configure the Timer A0 expansion register to divide the clock frequency by 1
    // Clear all bits of the EX0 register

    TIMER_A0->EX0 &= ~0xFF; // Clear all bits of the EX0 register - DSR

    // Configure the output mode as Toggle / Reset for CCR[3]
    // Set the bits of the OUTMOD field of the CCTL[3] register to 010b

    TIMER_A0->CCTL[3] |= 0x0040; // Setting CCTL Register to value of OUTMOD = 010b (Toggle/Reset), bits 7-5 -DSR


    // Assign the value of duty_cycle_1 to the CCR[3] register
    // Duty Cycle %: duty_cycle_1 / period_constant

    TIMER_A0->CCR[3] = duty_cycle_1; // Assigning value of duty_cycle_1 to the CCR[3] register - DSR


    // Configure the output mode as Toggle / Reset for CCR[4]
    // Set the bits of the OUTMOD field of the CCTL[4] register to 010b

    TIMER_A0->CCTL[4] |=  0x0040 ; // Setting CCTL Register to value of OUTMOD = 010b (Toggle/Reset), bits 7-5 -DSR-DSR

    // Assign the value of duty_cycle_1 to the CCR[4] register
    // Duty Cycle %: duty_cycle_2 / period_constant

    TIMER_A0->CCR[4] = duty_cycle_2; // Assigning value of duty_cycle_2 to the CCR[4] register - DSR

    // Modify the following bits in the CTL register
    // Select SMCLK = 12 MHz as timer clock source
    // Set ID = 3 (Divide timer clock by 8)
    // Set MC = 3 (Up/Down Mode)

    TIMER_A0->CTL = 0x0270 ; // See technical manual to see how values were selected - DSR

}


```

**Finalizing Motor.c Source Code**

The 2nd to last task was to complete the `Motor.c` source file, which would complete and implement the Motor Control Driver we can use to command the development board. This task involved completing the "Motor_Init", "Motor_Left", and "Motor_Right" functions. 


***Motor_Init Function***

The "Motor_Init" function was implemented by setting the motor control pins and starting the PWM timer signals. P5.4 and P5.5, which control motor direction, were configured as GPIO outputs and set low. P3.6 and P3.7, which enable the motors, were also configured as GPIO outputs and set low so the motors start off disabled. Finally, Timer A0 was initialized with a 20 ms period by calling Timer_A0_PWM_Init with a period constant of 15000.


```c 
void Motor_Init()
{
    // Configure the following pins as output GPIO pins: P5.4 and P5.5
    // by clearing Bits 4 and 5 of the SEL0 and SEL1 registers for P5
    // and setting Bits 4 and 5 of the DIR register for P5

    P5->SEL0 &= ~0x30; // Clearing Bits using Bit Mask 0011 0000-DSR
    P5->SEL1 &= ~0x30; // Clearing Bits using Bit Mask 0011 0000-DSR

    P5->DIR |= 0x30; // Setting Bits using Bit Mask 0011 0000-DSR


    // Initialize the output of the following pins to 0: P5.4 and P5.5
    // by clearing Bits 4 and 5 of the OUT register for P5

    P5->OUT &= ~0x30; // Clearing Bits using Bit Mask 0011 0000-DSR

    // Configure the following pins as output GPIO pins: P3.6 and P3.7
    // by clearing Bits 6 and 7 of the SEL0 and SEL1 registers for P3
    // and setting Bits 6 and 7 of the DIR register for P3

    P3->SEL0 &= ~0xC0; // Clearing Bits using Bit Mask 1100 0000-DSR
     P3->SEL1 &= ~0xC0; // Clearing Bits using Bit Mask 1100 0000-DSR

     P3->DIR |= 0xC0; // Setting Bits using Bit Mask 1100 0000-DSR


    // Initialize the output of the following pins to 0: P3.6 and P3.7
    // by clearing Bits 6 and 7 of the OUT register for P3

     P3->OUT &= ~0xC0; // Clearing Bits using Bit Mask 1100 0000-DSR

    // Initialize Timer A0 with a period of 20 ms using a period constant of 15000
    Timer_A0_PWM_Init(TIMER_A0_PERIOD_CONSTANT, 0, 0);
}
```


***Motor_Left Function***

The "Motor_Left" function is used to control the robot to turn left and was implemented by driving the two motors in opposite directions. The left motor was set to move backward by setting P5.4 high, and the right motor was set to move forward by clearing P5.5. The duty cycles for both motors were then updated using Timer_A0_Update_Duty_Cycle_1 and Timer_A0_Update_Duty_Cycle_2. Finally, both motors were enabled by setting P3.6 and P3.7 high.


```c
void Motor_Left(uint16_t left_duty_cycle, uint16_t right_duty_cycle)
{
    // Configure the left motor to move in a backward direction
    // by setting Bit 4 of the OUT register for P5

    P5->OUT |= 0x10; // Used Bit Mask 0001 0000 to set bit 4 -DSR


    // Configure the right motor to move in a forward direction
    // by clearing Bit 5 of the OUT register for P5

    P5->OUT &= ~0x20; // Used Bit Mask 00010 0000 to clear bit 5 -DSR

    // Update the duty cycle for both motors

    Timer_A0_Update_Duty_Cycle_1(right_duty_cycle); // Updating with the new duty cycle value -DSR
    Timer_A0_Update_Duty_Cycle_2(left_duty_cycle);  // Updating with the new duty cycle value -DSR


    // Enable the motors by setting Bits 6 and 7 of the OUT register for P3

    P3->OUT |= 0xC0; // Enabling Motor to allow them to perform this movement - DSR

}
```

***Motor_Right Function***

The "Motor_Right" function is used to control the robot to turn right and was implemented by driving the two motors in opposite directions. The left motor was set to move forward by clearing P5.4, and the right motor was set to move backward by setting P5.5. The duty cycles for both motors were then updated using Timer_A0_Update_Duty_Cycle_1 and Timer_A0_Update_Duty_Cycle_2. Finally, both motors were enabled by setting P3.6 and P3.7 high.

```c

void Motor_Right(uint16_t left_duty_cycle, uint16_t right_duty_cycle)
{
    // Configure the left motor to move in a forward direction
    // by clearing Bit 4 of the OUT register for P5

    P5->OUT &= ~0x10; // Used Bit Mask 0001 0000 to clear bit 4 -DSR

    // Configure the right motor to move in a backward direction
    // by setting Bit 5 of the OUT register for P5

    P5->OUT |= 0x20; // Used Bit Mask 0010 0000 to set bit 5 -DSR

    // Update the duty cycle for both motors

    Timer_A0_Update_Duty_Cycle_1(right_duty_cycle); // Updating with the new duty cycle value -DSR
    Timer_A0_Update_Duty_Cycle_2(left_duty_cycle);  // Updating with the new duty cycle value -DSR

    // Enable the motors by setting Bits 6 and 7 of the OUT register for P3

    P3->OUT |= 0xC0; // Enabling Motor to allow them to perform this movement - DSR

}

```


**Implementing Collision Detection**

The final task was to combine the bumper switch detection with the motor control to create a "Handle_Collision" function that will detect once the development board has run into something (i.e., drove into an obstacle/wall).

 Once a collision is detected, the motors are first stopped for two seconds. The robot then backs up for two seconds at a 30% duty cycle and stops for one second. Next, it turns right for four seconds at a 10% duty cycle and stops again for two seconds. Finally, the collision_detected flag is cleared to 0 so the robot is ready for the next collision. This can be seen below and found in `main.c`. 

***Handle_Collision Function***

```c
void Handle_Collision()
{
    // Stop the motors

    Motor_Stop();

    // Make a function call to Clock_Delay1ms(2000)

    Clock_Delay1ms(2000);

    // Move the motors backward with 30% duty cycle

    Motor_Backward(4500, 4500);

    // Make a function call to Clock_Delay1ms(2000)

    Clock_Delay1ms(2000);

    // Stop the motors

    Motor_Stop();

    // Make a function call to Clock_Delay1ms(1000)

    Clock_Delay1ms(1000);

    // Make the robot turn to the right with 10% duty cycle

    Motor_Right(15000 * 0.10, 15000 * 0.10);

    // Make a function call to Clock_Delay1ms(4000)

    Clock_Delay1ms(4000);

    // Stop the motors

    Motor_Stop();

    // Make a function call to Clock_Delay1ms(2000)

    Clock_Delay1ms(2000);

    // Set the collision_detected flag to 0

    collision_detected = 0;

}
```

To finalize this implementation, we also updated the "Bumper_Switches_Handler" function as well, so that if the collision global flag was not set and if the robot collides, it will set off the collision detection. This was implemented with the following code. 

***Bumper_Switches_Handler Function***

```c
void Bumper_Switches_Handler(uint8_t bumper_switch_state)
{
    if (collision_detected == 0)
    {
    printf("Collision Detected! Bumper Switch State: 0x%02X\n", bumper_switch_state);
    collision_detected = 1;

    }
}
```

Having all these functions implemented, to test out the functionality, we set the main while loop found in `main.c` to be the following. 

```c
    while(1)
    {
        // Rotate to 0
//        Timer_A2_Update_Duty_Cycle_1(1700);
//        Timer_A2_Update_Duty_Cycle_2(1700);
//        LED2_Output(RGB_LED_RED);
//        Clock_Delay1ms(3000);
//
//        // Rotate to 180
//        Timer_A2_Update_Duty_Cycle_1(7000);
//        Timer_A2_Update_Duty_Cycle_2(7000);
//        LED2_Output(RGB_LED_BLUE);
//        Clock_Delay1ms(3000);

//        Drive_Pattern_1();

        if (collision_detected == 1)
        {
            Handle_Collision();
        }
        else
        {
            Motor_Forward(4500, 4500);
        }
    }
```

---
### Known Issues or Limitations: 

During configuration of Timer A0, I initially set the CCTL\[3] and CCTL\[4] registers to a value of 2, reading the instruction to set OUTMOD to 010b as the value for the whole register. This was incorrect: 010b is the value for the OUTMOD field only, and that field sits in bits 7–5 of the 16-bit CCTL register. Writing 2 put the bit pattern in bits 2–0 instead, which set the COV flag and left OUTMOD at 000, so no PWM signal was produced on P2.6 and P2.7. Shifting the field value into its correct position (2 << 5 = 0x0040) set the output mode to Toggle/Reset as intended. I also found that the CTL register value of 0x0270 selected a clock divider of 2 rather than 8, and corrected it to 0x02F0 to get the specified 20 ms PWM period. 



---
### Author Contribution: 
Jamal B. helped with consulting and answering questions that came up during lab time. Code generated individually.
 
### References:

* MSP432P401R SimpleLink Microcontroller LaunchPad Development Kit User's Guide - [Link](https://docs.rs-online.com/3934/A700000006811369.pdf)

* MSP432P401R Datasheet - [Link](https://www.ti.com/lit/ds/slas826e/slas826e.pdf)

* MSP432P4xx SimpleLink Microcontrollers Technical Reference Manual - [Link](https://web.archive.org/web/20200402132841/http:/www.ti.com/lit/ug/slau356i/slau356i.pdf)