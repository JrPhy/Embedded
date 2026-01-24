在電腦中有許多的記憶體，如 CPU 中的快取記憶體，儲存設備的記憶體與應用程式開啟時使用的 RAM。在 MCU 中也有許多記憶體，例如 FLASH、EEPROM 與 RAM，每個部分使用到的記憶體也不一樣，特性也不一樣，其分配如下

|  | RAM | EEPROM | FLASH |
| --- | :---: | :---: | :---: |
| 揮發性 | 是 | 否 | 否 |
| 保持性 | 否 | 是 | 是 |
| 讀寫速度 | 快 | 慢 | 中 |
| 讀寫次數 | 多 | 中 | 少 |
| 成本 | 高 | 中 | 低 |
| 應用 | 開機後運算的資料 | 參數設定、小量資料 | 程式碼、韌體 |

## DMA
全名為直接記憶體存取（Direct Memory Access，DMA），是一種資料不須經過 CPU 就可以直接放進記憶體的方法。當資料一直傳輸時若需要 CPU 一直搬資料，那資源就會被占住，所以 CPU 只需要發何時開始接收資料，設備就會把資料寫進記憶體中，然後等資料全部放完或記憶體放滿後，CPU 再去讀取或是搬運資料，所以只需下幾次中斷，其餘時間就可以去做別的事情。
```c
#include "stm32f4xx.h" // 僅引入標頭檔以取得位址定義

void DMA_Config(uint32_t src, uint32_t dst, uint16_t size) {
    // 1. 開啟 DMA2 時鐘 (M2M 模式在 STM32F4 通常需要 DMA2)
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;

    // 2. 確保 DMA 通道已關閉，才能進行設定
    DMA2_Stream0->CR &= ~DMA_SxCR_EN;
    while(DMA2_Stream0->CR & DMA_SxCR_EN); // 等待硬體確認關閉

    // 3. 設定來源與目的地地址
    DMA2_Stream0->PAR = src;   // 周邊地址 (在 M2M 模式當作來源)
    DMA2_Stream0->M0AR = dst;  // 記憶體地址 (目的地)

    // 4. 設定搬運數量
    DMA2_Stream0->NDTR = size;

    // 5. 配置 CR 暫存器
    DMA2_Stream0->CR = 0; // 先清空
    DMA2_Stream0->CR |= (2 << 25);  // Channel 選擇 (視晶片手冊而定)
    DMA2_Stream0->CR |= DMA_SxCR_DIR_1; // 方向：記憶體到記憶體 (10)
    DMA2_Stream0->CR |= DMA_SxCR_MINC;  // 目的地地址遞增
    DMA2_Stream0->CR |= DMA_SxCR_PINC;  // 來源地址遞增 (M2M 時 PINC 代表來源)
    DMA2_Stream0->CR |= (1 << 13) | (1 << 11); // PSIZE 與 MSIZE 設為 16-bit (01)
    
    // 6. 啟動傳輸
    DMA2_Stream0->CR |= DMA_SxCR_EN;
}
```
