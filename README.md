# Wake-Up-Mate
#include "stm32f4xx_hal.h"
#include <cstring>
#include "mbed.h"
#include <cstdio>

// I2C and UART handlers
I2C_HandleTypeDef hi2c1;
UART_HandleTypeDef huart2;

// Motor and alarm components
#define MOTOR_PIN1 GPIO_PIN_0
#define MOTOR_PIN2 GPIO_PIN_1
#define MOTOR_PIN3 GPIO_PIN_2
#define MOTOR_PIN4 GPIO_PIN_3
#define MOTOR_PORT GPIOA
#define BUZZER_PIN GPIO_PIN_5
#define BUZZER_PORT GPIOB

// RTC Address (DS3231 or HW-054)
#define RTC_ADDRESS (0x68 << 1)  // Shift for HAL I2C

// Function prototypes
void SystemClock_Config(void);
void MX_GPIO_Init(void);
void MX_I2C1_Init(void);
void MX_USART2_UART_Init(void);
void Read_RTC_Time(void);
void Write_RTC_Time(uint8_t hours, uint8_t minutes, uint8_t seconds);
void Stepper_Move(uint8_t steps);
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart);
uint8_t BCD_To_Dec(uint8_t val);
uint8_t Dec_To_BCD(uint8_t val);

uint8_t rxBuffer[10];  // UART Receive Buffer
char message[50];

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C1_Init();
    MX_USART2_UART_Init();

    int length = snprintf(message, sizeof(message), "System Initialized\r\n");
    HAL_UART_Transmit(&huart2, (uint8_t*)message, length, HAL_MAX_DELAY);
    HAL_UART_Receive_IT(&huart2, rxBuffer, sizeof(rxBuffer));

    while (1) {
        Read_RTC_Time();
        Stepper_Move(8);  // Move 8 steps
        HAL_GPIO_TogglePin(BUZZER_PORT, BUZZER_PIN);
        HAL_Delay(1000);
    }
}

void Read_RTC_Time(void) {
    uint8_t timeData[3];
    HAL_I2C_Mem_Read(&hi2c1, RTC_ADDRESS, 0x00, I2C_MEMADD_SIZE_8BIT, timeData, 3, HAL_MAX_DELAY);

    uint8_t hours = BCD_To_Dec(timeData[2]);
    uint8_t minutes = BCD_To_Dec(timeData[1]);
    uint8_t seconds = BCD_To_Dec(timeData[0]);

    int length2 = snprintf(message, sizeof(message), "Time: %02d:%02d:%02d\r\n", hours, minutes, seconds);
    HAL_UART_Transmit(&huart2, (uint8_t*)message, length2, HAL_MAX_DELAY);
}

void Write_RTC_Time(uint8_t hours, uint8_t minutes, uint8_t seconds) {
    uint8_t timeData[3] = {Dec_To_BCD(seconds), Dec_To_BCD(minutes), Dec_To_BCD(hours)};
    HAL_I2C_Mem_Write(&hi2c1, RTC_ADDRESS, 0x00, I2C_MEMADD_SIZE_8BIT, timeData, 3, HAL_MAX_DELAY);
}

void Stepper_Move(uint8_t steps) {
    uint8_t step_sequence[4] = {0x09, 0x03, 0x06, 0x0C};
    for (int s = 0; s < steps; s++) {
        for (int i = 0; i < 4; i++) {
            HAL_GPIO_WritePin(MOTOR_PORT, MOTOR_PIN1, (step_sequence[i] & 0x01) ? GPIO_PIN_SET : GPIO_PIN_RESET);
            HAL_GPIO_WritePin(MOTOR_PORT, MOTOR_PIN2, (step_sequence[i] & 0x02) ? GPIO_PIN_SET : GPIO_PIN_RESET);
            HAL_GPIO_WritePin(MOTOR_PORT, MOTOR_PIN3, (step_sequence[i] & 0x04) ? GPIO_PIN_SET : GPIO_PIN_RESET);
            HAL_GPIO_WritePin(MOTOR_PORT, MOTOR_PIN4, (step_sequence[i] & 0x08) ? GPIO_PIN_SET : GPIO_PIN_RESET);
            HAL_Delay(10);
        }
    }
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART2) {
        uint8_t hours = (rxBuffer[0] - '0') * 10 + (rxBuffer[1] - '0');
        uint8_t minutes = (rxBuffer[2] - '0') * 10 + (rxBuffer[3] - '0');
        uint8_t seconds = (rxBuffer[4] - '0') * 10 + (rxBuffer[5] - '0');

        if (hours < 24 && minutes < 60 && seconds < 60) {
            Write_RTC_Time(hours, minutes, seconds);
        }

        HAL_UART_Receive_IT(&huart2, rxBuffer, sizeof(rxBuffer));
    }
}

uint8_t BCD_To_Dec(uint8_t val) {
    return ((val / 16) * 10) + (val % 16);
}

uint8_t Dec_To_BCD(uint8_t val) {
    return ((val / 10) * 16) + (val % 10);
}

void SystemClock_Config(void) {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLM = 8;
    RCC_OscInitStruct.PLL.PLLN = 336;
    RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
    RCC_OscInitStruct.PLL.PLLQ = 7;
    HAL_RCC_OscConfig(&RCC_OscInitStruct);
    
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK |
                                  RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5);
}

void MX_GPIO_Init(void) {
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = MOTOR_PIN1 | MOTOR_PIN2 | MOTOR_PIN3 | MOTOR_PIN4;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(MOTOR_PORT, &GPIO_InitStruct);

    GPIO_InitStruct.Pin = BUZZER_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    HAL_GPIO_Init(BUZZER_PORT, &GPIO_InitStruct);
}

void MX_I2C1_Init(void) {
    hi2c1.Instance = I2C1;
    hi2c1.Init.ClockSpeed = 100000;
    hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;
    hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    HAL_I2C_Init(&hi2c1);
}

void MX_USART2_UART_Init(void) {
    huart2.Instance = USART2;
    huart2.Init.BaudRate = 115200;
    HAL_UART_Init(&huart2);
}
