# PSC3M5 GPIO Interrupt-Based LED Control System

This project demonstrates how to use **GPIO interrupts** on the **Infineon PSC3M5 EVK** to control onboard LEDs using physical push buttons.  
It is a simple yet powerful example of handling asynchronous hardware events using the **Peripheral Driver Library (PDL)** of the PSC3 family.

---

## 🔧 Hardware Used
- **Infineon PSC3M5 Evaluation Kit (KIT_PSC3M5_EVK)**
- Onboard components:
  - **Yellow LED → P8_4**
  - **Blue LED → P8_5**
  - **Button BTN1 → P5_0 (Active High)**
  - **Button BTN2 → P0_4 (Active Low)**

---

## 🎯 Project Objective
- Configure two LEDs as **GPIO outputs**
- Configure two push buttons as **GPIO inputs**
- Enable **interrupts** on:
  - BTN1 → Rising edge interrupt
  - BTN2 → Falling edge interrupt
- Toggle LEDs when the corresponding button interrupt fires

This project showcases interrupt-driven I/O instead of traditional polling, enabling fast and efficient input response.

---

## 🧠 How It Works

### ▶ Button 1 (BTN1 - P5_0)
- Internally connected with a **pull-down resistor**
- Generates an **interrupt on rising edge**
- Toggles the **Yellow LED** (P8_4)

### ▶ Button 2 (BTN2 - P0_4)
- Internally connected with a **pull-up resistor**
- Generates an **interrupt on falling edge**
- Toggles the **Blue LED** (P8_5)

A flag-based mechanism ensures that LED toggling occurs in the main loop, keeping the interrupt service routine minimal.

---

## 🧩 Software Components Used
- **cybsp** – Board Support Package initialization
- **cy_pdl** – PSC3 Peripheral Driver Library
- **cy_device_headers** – Device definitions and register access

No HAL-CAT1 components are used because PSC3 devices require the **mtb-hal-psc3** HAL.

---

## 📄 Source Code (main.c)

```c
#include "cybsp.h"
#include "cy_device_headers.h"
#include "cy_pdl.h"
#include <stdbool.h>

/* LED Pins (from schematic) */
#define LED_YELLOW_PORT     GPIO_PRT8
#define LED_YELLOW_PIN      4
#define LED_BLUE_PORT       GPIO_PRT8
#define LED_BLUE_PIN        5

/* Button Pins (PSC3 EVK Board) */
#define BTN1_PORT           GPIO_PRT5   // Active High
#define BTN1_PIN            0
#define BTN2_PORT           GPIO_PRT0   // Active Low
#define BTN2_PIN            4

volatile bool btn1_flag = false;
volatile bool btn2_flag = false;

/* BTN1 ISR – Rising edge */
void btn1_isr(void)
{
    Cy_GPIO_ClearInterrupt(BTN1_PORT, BTN1_PIN);
    btn1_flag = true;
}

/* BTN2 ISR – Falling edge */
void btn2_isr(void)
{
    Cy_GPIO_ClearInterrupt(BTN2_PORT, BTN2_PIN);
    btn2_flag = true;
}

int main(void)
{
    cy_rslt_t result = cybsp_init();
    if (result != CY_RSLT_SUCCESS) { CY_ASSERT(0); }

    __enable_irq();

    /* Configure LEDs */
    Cy_GPIO_Pin_FastInit(LED_YELLOW_PORT, LED_YELLOW_PIN,
                         CY_GPIO_DM_STRONG, 0, HSIOM_SEL_GPIO);
    Cy_GPIO_Pin_FastInit(LED_BLUE_PORT, LED_BLUE_PIN,
                         CY_GPIO_DM_STRONG, 0, HSIOM_SEL_GPIO);

    /* Configure BTN1 - rising edge */
    Cy_GPIO_Pin_FastInit(BTN1_PORT, BTN1_PIN,
                         CY_GPIO_DM_PULLDOWN, 0, HSIOM_SEL_GPIO);
    Cy_GPIO_SetInterruptEdge(BTN1_PORT, BTN1_PIN, CY_GPIO_INTR_RISING);
    Cy_GPIO_ClearInterrupt(BTN1_PORT, BTN1_PIN);
    
    Cy_SysInt_Init(&((cy_stc_sysint_t){
        .intrSrc = ioss_interrupts_gpio_5_IRQn,
        .intrPriority = 3 }),
        btn1_isr);
    NVIC_EnableIRQ(ioss_interrupts_gpio_5_IRQn);

    /* Configure BTN2 - falling edge */
    Cy_GPIO_Pin_FastInit(BTN2_PORT, BTN2_PIN,
                         CY_GPIO_DM_PULLUP, 1, HSIOM_SEL_GPIO);
    Cy_GPIO_SetInterruptEdge(BTN2_PORT, BTN2_PIN, CY_GPIO_INTR_FALLING);
    Cy_GPIO_ClearInterrupt(BTN2_PORT, BTN2_PIN);

    Cy_SysInt_Init(&((cy_stc_sysint_t){
        .intrSrc = ioss_interrupts_gpio_0_IRQn,
        .intrPriority = 3 }),
        btn2_isr);
    NVIC_EnableIRQ(ioss_interrupts_gpio_0_IRQn);

    /* Main Loop */
    for (;;)
    {
        if (btn1_flag)
        {
            btn1_flag = false;
            Cy_GPIO_Inv(LED_YELLOW_PORT, LED_YELLOW_PIN);
        }

        if (btn2_flag)
        {
            btn2_flag = false;
            Cy_GPIO_Inv(LED_BLUE_PORT, LED_BLUE_PIN);
        }
    }
}
