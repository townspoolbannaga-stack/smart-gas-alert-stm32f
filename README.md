01  #include <stdint.h>

02  /* ================= BASE ADDRESSES ================= */
03  #define PERIPH_BASE     0x40000000UL
04  #define AHB1_BASE       (PERIPH_BASE + 0x00020000UL)
05  #define APB2_BASE       (PERIPH_BASE + 0x00010000UL)

06  /* ================= RCC ================= */
07  #define RCC_BASE        (AHB1_BASE + 0x3800UL)
08  #define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30))
09  #define RCC_APB2ENR     (*(volatile uint32_t *)(RCC_BASE + 0x44))

10  /* ================= GPIO ================= */
11  #define GPIOA_BASE      (AHB1_BASE + 0x0000UL)
12  #define GPIOB_BASE      (AHB1_BASE + 0x0400UL)
13  #define GPIOD_BASE      (AHB1_BASE + 0x0C00UL)

14  #define GPIOA_MODER     (*(volatile uint32_t *)(GPIOA_BASE + 0x00))
15  #define GPIOB_MODER     (*(volatile uint32_t *)(GPIOB_BASE + 0x00))
16  #define GPIOD_MODER     (*(volatile uint32_t *)(GPIOD_BASE + 0x00))
17  #define GPIOB_ODR       (*(volatile uint32_t *)(GPIOB_BASE + 0x14))
18  #define GPIOD_ODR       (*(volatile uint32_t *)(GPIOD_BASE + 0x14))

19  /* ================= ADC ================= */
20  #define ADC1_BASE       (APB2_BASE + 0x2000UL)
21  #define ADC_SR          (*(volatile uint32_t *)(ADC1_BASE + 0x00))
22  #define ADC_CR1         (*(volatile uint32_t *)(ADC1_BASE + 0x04))
23  #define ADC_CR2         (*(volatile uint32_t *)(ADC1_BASE + 0x08))
24  #define ADC_SMPR2       (*(volatile uint32_t *)(ADC1_BASE + 0x10))
25  #define ADC_SQR3        (*(volatile uint32_t *)(ADC1_BASE + 0x34))
26  #define ADC_DR          (*(volatile uint32_t *)(ADC1_BASE + 0x4C))

27  /* ================= NVIC ================= */
28  #define NVIC_ISER0      (*(volatile uint32_t *)0xE000E100)

29  /* ================= GLOBAL VARIABLES ================= */
30  volatile uint16_t gas_value = 0;
31  const uint16_t GAS_THRESHOLD = 1500;

32  /* ================= ADC INTERRUPT HANDLER ================= */
33  void ADC_IRQHandler(void)
34  {
35      if (ADC_SR & (1 << 1))   // EOC flag
36      {
37          gas_value = ADC_DR;  // Read ADC value (clears EOC)

38          if (gas_value > GAS_THRESHOLD)
39          {
40              /* LEDs ON */
41              GPIOB_ODR &= ~(0xF << 12);

42              /* Motor OFF */
43              GPIOB_ODR &= ~(1 << 5);
44              GPIOB_ODR &= ~(1 << 6);
45              GPIOB_ODR &= ~(1 << 7);

46              /* Buzzer OFF */
47              GPIOD_ODR &= ~(1 << 2);
48          }
49          else
50          {
51              /* LEDs OFF */
52              GPIOB_ODR |= (0xF << 12);

53              /* Motor ON */
54              GPIOB_ODR |=  (1 << 5);
55              GPIOB_ODR |=  (1 << 6);
56              GPIOB_ODR &= ~(1 << 7);

57              /* Buzzer ON */
58              GPIOD_ODR |= (1 << 2);
59          }
60      }
61  }

62  /* ================= MAIN ================= */
63  int main(void)
64  {
65      /* Enable clocks */
66      RCC_AHB1ENR |= (1 << 0);  // GPIOA
67      RCC_AHB1ENR |= (1 << 1);  // GPIOB
68      RCC_AHB1ENR |= (1 << 3);  // GPIOD
69      RCC_APB2ENR |= (1 << 8);  // ADC1

70      /* GPIO configuration */

71      /* PA1 - Analog */
72      GPIOA_MODER |= (3 << (1 * 2));

73      /* PD2 - Buzzer */
74      GPIOD_MODER |= (1 << (2 * 2));

75      /* PB5,6,7 - Motor */
76      GPIOB_MODER |= (1 << (5 * 2));
77      GPIOB_MODER |= (1 << (6 * 2));
78      GPIOB_MODER |= (1 << (7 * 2));

79      /* PB12–PB15 - LEDs */
80      GPIOB_MODER |= (0x55 << (12 * 2));

81      /* ADC configuration */
82      ADC_SMPR2 |= (7 << 3);    // Long sample time CH1
83      ADC_SQR3 = 1;             // Channel 1 (PA1)
84      ADC_CR1 |= (1 << 5);      // EOC interrupt enable
85      ADC_CR2 |= (1 << 0);      // ADC ON

86      /* Enable ADC interrupt in NVIC (ADC IRQ = 18) */
87      NVIC_ISER0 |= (1 << 18);

88      while (1)
89      {
90          ADC_CR2 |= (1 << 30);  // Start ADC conversion
91          /* CPU does NOT wait here */
92      }  
93  }


