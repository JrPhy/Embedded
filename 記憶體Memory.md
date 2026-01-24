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
#include "stm32f4xx.h"

#define BUFFER_SIZE 32
uint32_t src_buffer[BUFFER_SIZE];
uint32_t dst_buffer[BUFFER_SIZE];
volatile uint8_t transfer_complete = 0; // 必須加 volatile

void DMA_M2M_Init_With_IT(void) {
    // 1. 開啟 DMA2 時鐘
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;

    // 2. 確保 Stream 停用以進行配置
    DMA2_Stream0->CR &= ~DMA_SxCR_EN;
    while(DMA2_Stream0->CR & DMA_SxCR_EN);

    // 3. 設定來源與目的地地址
    DMA2_Stream0->PAR  = (uint32_t)src_buffer; // 來源
    DMA2_Stream0->M0AR = (uint32_t)dst_buffer; // 目的地
    DMA2_Stream0->NDTR = BUFFER_SIZE;          // 搬運數量
    // 4. 配置控制暫存器 (CR)
    // M2M=10 (記憶體到記憶體), PINC=1, MINC=1, PSIZE/MSIZE=10 (32-bit)
    DMA2_Stream0->CR = 0; 
    DMA2_Stream0->CR |= (2 << 6)  | // DIR: Memory-to-memory
                        (1 << 10) | // MINC: 目的地地址遞增
                        (1 << 9)  | // PINC: 來源地址遞增
                        (2 << 13) | // PSIZE: 32-bit
                        (2 << 11) | // MSIZE: 32-bit
                        (1 << 4);   // TCIE: 開啟傳輸完成中斷
    // 5. NVIC 中斷控制器設定
    NVIC_SetPriority(DMA2_Stream0_IRQn, 0);
    NVIC_EnableIRQ(DMA2_Stream0_IRQn);
}

// --- 中斷服務程式 ---
void DMA2_Stream0_IRQHandler(void) {
    // 檢查 DMA2 Stream 0 的傳輸完成標誌 (TCIF0)
    if (DMA2->LISR & DMA_LISR_TCIF0) {
        transfer_complete = 1; // 設置軟體標誌位
        // 重要：手動清除中斷標誌位，否則會無限觸發
        DMA2->LIFCR |= DMA_LIFCR_CTCIF0;
    }
}

int main(void) {
    // 初始化緩衝區資料
    for(int i=0; i<BUFFER_SIZE; i++) src_buffer[i] = i;

    DMA_M2M_Init_With_IT();

    // 啟動 DMA 傳輸
    DMA2_Stream0->CR |= DMA_SxCR_EN;

    while(1) {
        if (transfer_complete) {
            // 這裡處理搬運完成後的逻辑
            transfer_complete = 0;
            // 可以在此再次開啟傳輸或做其他事
        }
    }
}
```
