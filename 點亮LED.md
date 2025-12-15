使用不同的單晶片點亮 LED 的方式都不同，這取決於開發商幫你做掉了多少。

## 1. STM32
需要去設置 GPIO port 的狀態以及頻率 https://www.cnblogs.com/zxr-blog/p/17957466#_lab2_0_0
```C
#include "led.h"
void main(void) {
    GPIO_InitTypeDef GPIO_InitStruct;//结构图定义
    RCC_APB2PeriphClockCmd(LED_G_CLK,ENABLE);//总线2时钟开关
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_1;//引脚
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_PP;//模式为推挽输出
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;//设置频率为50HZ
    GPIO_Init(LED_G_PORT,  &GPIO_InitStruct);//GPIO初始化
    while (1) {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);   // 點亮 LED
        HAL_Delay(1000);                                      // 延遲 1 秒
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET); // 熄滅 LED
        HAL_Delay(1000);                                      // 延遲 1 秒
    }
}
```

## 2. 8051
8051 的每個 port 都是預設高電位，所以只要將連接的 port 拉低電位即可點亮
```C
#include <reg51.h>
sbit LED = P1^0; // 定義 LED 連接到 P1.0
void main() {
    while(1) {
        LED = 1; // 點亮 LED
        delay(1000);
        LED = 0; // 熄滅 LED
        delay(1000);
    }
}

```

## 3. 裸機
```C
#define LED_PIN  (1 << 5) // 假設 LED 接在第 5 腳
void led_on() {
    GPIO_PORT |= LED_PIN; // 點亮 LED
}
void led_off() {
    GPIO_PORT &= ~LED_PIN; // 熄滅 LED
}
```
