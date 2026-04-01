# ARM Cortex M3 | STM32F100 | Embedded Systems  
  
## Worksheet 3 — GPIO, Button Input, and FSM Design
  
**Student Name:** Joby Barnaby 
**Student ID:** 24049911

---
## Overview

This worksheet covers programming GPIO on the STM32F100 microcontroller to control LEDs and read button input. It progresses through five exercises, finishing with a Finite State Machine (FSM) implementation for reliable debounced button handling.
### Hardware Used

| Component   | Pin | Description   |
| ----------- | --- | ------------- |
| Green LED   | PC6 | Port C, Pin 6 |
| Yellow LED  | PC7 | Port C, Pin 7 |
| WKUP Button | PA0 | Port A, Pin 0 |

---

## Repository Setup


```bash

cd ~/SES

git clone https://gitlab.uwe.ac.uk/c-duffy/ses-worksheet-3.git

cd ses-worksheet-3/worksheet3

```

  

---
## File Structure


```

ses-worksheet-3/

├── STM32F10x_StdPeriph_Lib_V3.5.0/

│   └── Libraries/STM32F10x_StdPeriph_Driver/

│       ├── src/

│       │   ├── stm32f10x_rcc.c

│       │   └── stm32f10x_gpio.c

│       └── inc/

│           ├── stm32f10x_rcc.h

│           └── stm32f10x_gpio.h

└── worksheet3/

    ├── main.c               ← Your code

    ├── Makefile             ← Modified to include RCC + GPIO

    ├── openocd.cfg

    ├── startup_stm32f10x.c

    ├── stm32f100.ld

    └── stm32f10x_conf.h

```

  
---

## Makefile Changes

  

Two changes were made to the `Makefile` before starting:

  

**1. Add GPIO and RCC library objects:**

```makefile

# Before

OBJS= $(STARTUP) main.o

  

# After

OBJS= $(STARTUP) stm32f10x_rcc.o stm32f10x_gpio.o main.o

```

  

**2. Disable compiler optimisation (for reliable debugging):**

```makefile

# Before

CFLAGS = -O1 -g

  

# After

CFLAGS = -O0 -g

```

![makefile](w3screenshot/1.png)

---

  

## Build & Flash Workflow

Every exercise follows the same build and flash process:

**Build commands:**

```bash

make clean

make
```

**Terminal 1 — Start OpenOCD:**

```bash

openocd -f openocd.cfg

```

**Terminal 2 — Flash the board:**

```bash

telnet localhost 4444

reset halt

flash write_image erase demo.elf 0

reset run

```

---

## Exercise 1 — Flash Green LED

**Description:** 
This exercise demonstrates basic GPIO output by flashing an LED connected to PC6 using a software delay loop

**Code:** 
```c
#include <stm32f10x.h>  
#include <stm32f10x_rcc.h>  
#include <stm32f10x_gpio.h>

GPIO_InitTypeDef GPIO_InitStructure;

int main(void)  
{  
    volatile uint32_t i;  
    static int greenledval = 0;  
     
    /* Enable GPIOC clock */  
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);  
     
    /* Configure PC6 (Green LED) as output push-pull */  
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;  
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;  
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;  
    GPIO_Init(GPIOC, &GPIO_InitStructure);  
     
    /* Main loop - flash LED */  
    while (1)  
    {  
        GPIO_WriteBit(GPIOC, GPIO_Pin_6, (greenledval) ? Bit_SET : Bit_RESET);  
        greenledval = 1 - greenledval;  
         
        /* Delay */  
        i = 1000000;  
        while(--i);  
    }  
}
#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t* file, uint32_t line)
{
    while (1)
    {
        /* Infinite loop for debugging */
    }
}
#endif
```

**Explanation:** 
The GPIO pin PC6 is configured as a push-pull output. the program continuously toggles the pin between HIGH and LOW, causing the LED to flash. A delay loop is used to slow down the flashing so it is visible.

**Result:** 
The green LED flashes on and off continuously, demonstrating basic GPIO output control.

![Exercise 1](w3screenshot/2.png)

![Watch the demo](video/1.mov)

---

## Exercise 2 — Alternating Green and Yellow LEDs

**Description:** 
This exercise extends the previous task by controlling two LEDs (PC6 & PC7) and alternating between them. 

**Code:** 
```c
#include <stm32f10x.h>
#include <stm32f10x_rcc.h>
#include <stm32f10x_gpio.h>

GPIO_InitTypeDef GPIO_InitStructure;

int main(void)

{

    volatile uint32_t i;

    static int ledstate = 0;

    /* Enable GPIOC clock */

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

    /* Configure PC6 (Green) and PC7 (Yellow) as output */

    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_6 | GPIO_Pin_7;

    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_Out_PP;

    GPIO_Init(GPIOC, &GPIO_InitStructure);

    while (1)

    {

        if (ledstate == 0) {

            GPIO_WriteBit(GPIOC, GPIO_Pin_6, Bit_SET);

            GPIO_WriteBit(GPIOC, GPIO_Pin_7, Bit_RESET);

        } else {

            GPIO_WriteBit(GPIOC, GPIO_Pin_6, Bit_RESET);

            GPIO_WriteBit(GPIOC, GPIO_Pin_7, Bit_SET);

        }

        ledstate = 1 - ledstate;

        i = 2000000;

        while(--i);

    }

}

#ifdef USE_FULL_ASSERT

void assert_failed(uint8_t* file, uint32_t line)

{

    while (1) {}

}

#endif
```

**Explanation:** 
Both LEDs are configured as outputs. the program alternates which LED is active by switching one HIGH and the other LOW in each loop interanion. 

**Result:** 
The green and yellow LEDs alternate, demonstrating control of multiple outputs.

![exercise 2](w3screenshot/3.png)

![Watch the demo](video/2.mov)

---
## Exercise 3 — Button Controls LED

**Description:** 
This exercise introduces input handling. A button connected to PA0 is used to control an LED on PC6 

**Code:** 
```c
#include <stm32f10x.h>
#include <stm32f10x_rcc.h>
#include <stm32f10x_gpio.h>
  
GPIO_InitTypeDef GPIO_InitStructure;

  

int main(void) {

    uint8_t button_state;

  

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

  

    /* Configure PC6 as output */

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;

    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;

    GPIO_Init(GPIOC, &GPIO_InitStructure);

  

    /* Configure PA0 as floating input */

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;

    GPIO_Init(GPIOA, &GPIO_InitStructure);

  

    while (1) {

        button_state = GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0);

  

        if (button_state == 1) {

            GPIO_WriteBit(GPIOC, GPIO_Pin_6, Bit_SET);

        } else {

            GPIO_WriteBit(GPIOC, GPIO_Pin_6, Bit_RESET);

        }

    }

}

  

#ifdef USE_FULL_ASSERT

void assert_failed(uint8_t* file, uint32_t line) {

    while (1) {}

}

#endif

```

**Explanation:** 
The button input is read continuously. when the button is pressed, the LED is turned on. When released, the LED is turned off. this demonstrates direct mapping between input and output.

**Result:** 
The LED turns on while the button is pressed and off when released.

![Exercise 3](w3screenshot/4.png)

![Watch the demo](video/3.mov)

---

## Exercise 4 — Toggle LED with Button

**Description:** 
This exercise introduces edge detection. The LED toggles state each time the button is pressed.

**Code:** 
```c

#include <stm32f10x.h>
#include <stm32f10x_rcc.h>
#include <stm32f10x_gpio.h>

GPIO_InitTypeDef GPIO_InitStructure;

int main(void) {

    uint8_t button_state;

    uint8_t last_button_state = 0;

    uint8_t led_state = 0;

  

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

  

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;

    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;

    GPIO_Init(GPIOC, &GPIO_InitStructure);

  

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;

    GPIO_Init(GPIOA, &GPIO_InitStructure);

  

    while (1) {

        button_state = GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0);

  

        /* Detect rising edge (0 -> 1 transition) */

        if (button_state == 1 && last_button_state == 0) {

            led_state = 1 - led_state;

        }

  

        last_button_state = button_state;

        GPIO_WriteBit(GPIOC, GPIO_Pin_6, led_state ? Bit_SET : Bit_RESET);

    }

}
#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t* file, uint32_t line) {
    while (1) {}
}
#endif

```

**Explanation:** 
The program detects a rising edge (transition from not pressed to pressed) to toggle the LED state. this prevents the LED from rapidly switching while the button is held down.

**Result:**
Each button press toggles the LED. However due to mechanical bouncing, multiple toggles may occur per press. 

![exercise 4](w3screenshot/4.png)

![Watch the demo](video/4.mov)

---

  

## Exercise 5 — Debounced Toggle

**Description:** 
This exercise improves the button-controlled LED toggle by implementing a software debounce algorithm. this prevents multiple false triggers caused by mechanical switch bouncing.
  
**Code:** 
```c

#include <stm32f10x.h>
#include <stm32f10x_rcc.h>
#include <stm32f10x_gpio.h>

GPIO_InitTypeDef GPIO_InitStructure;

#define DEBOUNCE_COUNT 50

  

int main(void) {

    uint8_t  button_state;

    uint8_t  stable_state   = 0;

    uint8_t  led_state      = 0;

    uint16_t press_counter  = 0;

    uint16_t release_counter = 0;

  

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

  

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;

    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;

    GPIO_Init(GPIOC, &GPIO_InitStructure);

  

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;

    GPIO_Init(GPIOA, &GPIO_InitStructure);

  

    while (1) {

        button_state = GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0);

  

        if (button_state == 1) {

            press_counter++;

            release_counter = 0;

  

            if (press_counter >= DEBOUNCE_COUNT && stable_state == 0) {

                stable_state = 1;

                led_state = 1 - led_state;

            }

        } else {

            release_counter++;

            press_counter = 0;

  

            if (release_counter >= DEBOUNCE_COUNT && stable_state == 1) {

                stable_state = 0;

            }

        }

  

        GPIO_WriteBit(GPIOC, GPIO_Pin_6, led_state ? Bit_SET : Bit_RESET);

    }

}
#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t* file, uint32_t line) {
    while (1) {}
}
#endif

```

**Explanation:** 
Mechanical buttons do not produce clean transitions when pressed or released. instead they rapidly switch between HIGH and LOW for a short period, which can cause multiple false triggers.

to solve this, a counter-based debounce method is used. the button must remain in the same state for a fixed number of consecutive readings (`DEBOUNCE_COUNT`) before the input is considered stable.

- when the button is pressed, a counter increments until it reaches the threshold.
- once confirmed, the LED state is toggled.
- A similar process is used to confirm when the button is released.

This ensure that only deliberate pressed result in a state change.

**Result:** 
The LED toggles reliably with each button press. unlike the previous exercise, there are no unintended multiple toggles cause by switch bounce 

![Exercise 5](w3screenshot/6.png)

![Watch the demo](video/5.mov)

---

  

## Extra Task — Alternating LEDs on Button Press (Debounced)

  **Description:**
  This task extends the debounced button logic from Exercise 5 to control two LEDs. Each valid button press alternates between the green LED (PC6) and the yellow LED (PC7).

  **Code:**
```c

#include <stm32f10x.h>
#include <stm32f10x_rcc.h>
#include <stm32f10x_gpio.h>

GPIO_InitTypeDef GPIO_InitStructure;

#define DEBOUNCE_COUNT 50
  
int main(void) {

    uint8_t  button_state;

    uint8_t  stable_state    = 0;

    uint8_t  led_state       = 0;

    uint16_t press_counter   = 0;

    uint16_t release_counter = 0;

  

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

  

    /* Configure PC6 (Green) and PC7 (Yellow) as outputs */

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;

    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;

    GPIO_Init(GPIOC, &GPIO_InitStructure);

  

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;

    GPIO_Init(GPIOA, &GPIO_InitStructure);

  

    /* Start: Green ON, Yellow OFF */

    GPIO_WriteBit(GPIOC, GPIO_Pin_6, Bit_SET);

    GPIO_WriteBit(GPIOC, GPIO_Pin_7, Bit_RESET);

  

    while (1) {

        button_state = GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0);

  

        if (button_state == 1) {

            press_counter++;

            release_counter = 0;

  

            if (press_counter >= DEBOUNCE_COUNT && stable_state == 0) {

                stable_state = 1;

                led_state = 1 - led_state;

  

                if (led_state == 0) {

                    GPIO_WriteBit(GPIOC, GPIO_Pin_6, Bit_SET);

                    GPIO_WriteBit(GPIOC, GPIO_Pin_7, Bit_RESET);

                } else {

                    GPIO_WriteBit(GPIOC, GPIO_Pin_6, Bit_RESET);

                    GPIO_WriteBit(GPIOC, GPIO_Pin_7, Bit_SET);

                }

            }

        } else {

            release_counter++;

            press_counter = 0;

  

            if (release_counter >= DEBOUNCE_COUNT && stable_state == 1) {

                stable_state = 0;

            }

        }

    }

}

#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t* file, uint32_t line) {
    while (1) {}
}
#endif

```

**Explanation:**
This program builds on the debounce logic by introducing two LEDs. A variable (`led_state`)  is used to track which LED should be active.

Each valid button press (after debouncing using `DEBOUNCE_COUNT`) toggles the state, switching between the green and yellow LEDs. this demonstrates how debounced input can be used to control more complex output behaviour 

**Result:**
Each button press alternates between the green and yellow LEDs. The debounce logic ensures that only one toggles occurs per press, eliminating false triggers. 

![Extra task](w3screenshot/7.png)

![Watch the demo](video/6.mov)

---

  

## Merit Task — FSM Debounced Button with Alternating LEDs

**Description:**
This task refactors the debounce logic into a Finite State Machine (FSM). This approach separates the button handling process into distinct states, improving code clarity, reliability, and maintainability.

 **FSM State Diagram**

```

         button == 1                    count >= 50

  IDLE ─────────────► DEBOUNCE_PRESS ─────────────► PRESSED

   ▲                       │                            │

   │         button == 0   │              button == 0   │

   │         (noise)       ▼                            ▼

   │                     IDLE          DEBOUNCE_RELEASE

   │                                         │

   └─────────────────────────────────────────┘

              count >= 50 (confirmed release)

```


**Code:**   

```c

#include <stm32f10x.h>
#include <stm32f10x_rcc.h>
#include <stm32f10x_gpio.h>

GPIO_InitTypeDef GPIO_InitStructure;

typedef enum {

    STATE_IDLE             = 0,

    STATE_DEBOUNCE_PRESS   = 1,

    STATE_PRESSED          = 2,

    STATE_DEBOUNCE_RELEASE = 3

} ButtonState;

  

#define DEBOUNCE_COUNT 50

  

int main(void) {

    ButtonState current_state    = STATE_IDLE;

    uint16_t    debounce_counter = 0;

    uint8_t     led_state        = 0;

    uint8_t     button_reading;

  

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

  

    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_6 | GPIO_Pin_7;

    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_Out_PP;

    GPIO_Init(GPIOC, &GPIO_InitStructure);

  

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_0;

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;

    GPIO_Init(GPIOA, &GPIO_InitStructure);

  

    /* Start: Green ON, Yellow OFF */

    GPIOC->BSRR = 1 << 6;

    GPIOC->BRR  = 1 << 7;

  

    while (1) {

        button_reading = GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0);

  

        switch (current_state)

        {

            case STATE_IDLE:

                if (button_reading == 1) {

                    debounce_counter = 0;

                    current_state = STATE_DEBOUNCE_PRESS;

                }

                break;

  

            case STATE_DEBOUNCE_PRESS:

                if (button_reading == 1) {

                    debounce_counter++;

                    if (debounce_counter >= DEBOUNCE_COUNT) {

                        /* Confirmed press — toggle LEDs */

                        led_state = 1 - led_state;

                        if (led_state == 1) {

                            GPIOC->BRR  = 1 << 6;   /* Green OFF */

                            GPIOC->BSRR = 1 << 7;   /* Yellow ON */

                        } else {

                            GPIOC->BSRR = 1 << 6;   /* Green ON  */

                            GPIOC->BRR  = 1 << 7;   /* Yellow OFF */

                        }

                        debounce_counter = 0;

                        current_state = STATE_PRESSED;

                    }

                } else {

                    /* Noise — abort and return to IDLE */

                    debounce_counter = 0;

                    current_state = STATE_IDLE;

                }

                break;

  

            case STATE_PRESSED:

                if (button_reading == 0) {

                    debounce_counter = 0;

                    current_state = STATE_DEBOUNCE_RELEASE;

                }

                break;

  

            case STATE_DEBOUNCE_RELEASE:

                if (button_reading == 0) {

                    debounce_counter++;

                    if (debounce_counter >= DEBOUNCE_COUNT) {

                        debounce_counter = 0;

                        current_state = STATE_IDLE;

                    }

                } else {

                    /* Noise — still pressed */

                    debounce_counter = 0;

                    current_state = STATE_PRESSED;

                }

                break;

  

            default:

                current_state = STATE_IDLE;

                break;

        }

    }

}

#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t* file, uint32_t line) {
    while (1) {}
}
#endif

```

**Explanation:**
A Finite State Machine (FSM) is used to structure the button handling logic into clearly defined states:

- `STATE_IDLE`: Waiting for a button press
- `STATE_DEBOUNCE_PRESS`: Verifying a stable press
- `STATE_PRESSED`: Press confirmed
- `STATE_DEBOUNCE_RELEASE`: Verifying release

Each state handles a specific part of the input process, making the system easier to understand and extend. The debounce logic is integrated into the state transitions using a counter (`DEBOUNCE_COUNT`).

**Result:** 
Each button press reliably alternates between the green and yellow LEDs. The FSM structure ensures clean transitions and eliminates false triggers caused by switch bounce.

![merit](w3screenshot/8.png)

![Watch the demo](video/6.mov)

---
## Key Concepts

### Why enable the clock first?

The STM32 saves power by keeping peripheral clocks gated off by default. `RCC_APB2PeriphClockCmd()` must be called before any GPIO register can be written, otherwise the peripheral is unresponsive.

  

### GPIO Modes Used

| Mode               | Constant                  | Description                          |
|--------------------|--------------------------|--------------------------------------|
| Output Push-Pull   | `GPIO_Mode_Out_PP`       | Outputs a digital HIGH or LOW to drive components such as LEDs |
| Input Floating     | `GPIO_Mode_IN_FLOATING`  | Reads input signals without internal pull resistors (external circuit required) |

### What is switch bouncing?

Mechanical buttons don't make clean transitions — the contacts physically vibrate for a few milliseconds, causing rapid 0/1 noise around each press and release. Without debouncing, a single press can register as many events.

### Counter debounce vs FSM debounce

The counter approach (Exercise 5) works correctly but mixes the press and release logic in one block. The FSM approach (Merit task) separates each phase into its own state, making the code easier to read, test, and modify.

---
## Final Conclusion

In this worksheet, I successfully developed a series of programs to control LEDs and read button input using the STM32 microcontroller. The tasks progressed from simple GPIO output, such as flashing an LED, to more advanced input handling techniques including edge detection and debouncing.

The implementation of a counter-based debounce algorithm significantly improved the reliability of button input by eliminating false triggers caused by mechanical switch bounce. This was further enhanced by refactoring the logic into a finite state machine (FSM), which provided a clearer and more structured approach to handling input states.

Overall, this worksheet demonstrated key embedded systems concepts, including GPIO configuration, real-time input/output interaction, and state-based system design. A possible improvement would be to replace software delays with hardware timers or interrupts to improve efficiency and responsiveness.

