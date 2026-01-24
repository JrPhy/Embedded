在電腦中有許多的記憶體，如 CPU 中的快取記憶體，儲存設備的記憶體與應用程式開啟時使用的 RAM。在 MCU 中也有許多記憶體，例如 FLASH、EEPROM 與 RAM，每個部分使用到的記憶體也不一樣，特性也不一樣，其分配如下
## 一、記憶體種類
|  | RAM | EEPROM | FLASH |
| --- | :---: | :---: | :---: |
| 揮發性 | 是 | 否 | 否 |
| 保持性 | 否 | 是 | 是 |
| 讀寫速度 | 快 | 慢 | 中 |
| 讀寫次數 | 多 | 中 | 少 |
| 成本 | 高 | 中 | 低 |
| 應用 | 開機後運算的資料 | 參數設定、小量資料 | 程式碼、韌體 |

## 二、DMA
全名為直接記憶體存取（Direct Memory Access，DMA），是一種資料不須經過 CPU 就可以直接放進記憶體的方法。當資料一直傳輸時若需要 CPU 一直搬資料，那資源就會被占住，所以 CPU 只需要發何時開始接收資料，設備就會把資料寫進記憶體中，然後等資料全部放完或記憶體放滿後，CPU 再去讀取或是搬運資料，所以只需下幾次中斷，其餘時間就可以去做別的事情。
#### 1. 單緩衝模式 Single Buffer
在一些記憶體有限的情況，或是只需要搬運很少的量時就使用單緩衝模式，例如 IOT 裝置，開機時的初始化等就可以使用此模式
```c
#include "stm32f4xx.h"

#define BUF_SIZE 64
uint8_t src[BUF_SIZE], dst[BUF_SIZE];

void DMA_M2M_Init_With_IT(void) {
    // 1. 時鐘配置
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;

    // 2. 停止 Stream 並等待硬體就緒
    DMA2_Stream0->CR &= ~DMA_SxCR_EN;
    while(DMA2_Stream0->CR & DMA_SxCR_EN);

    // 3. 基本搬運參數
    DMA2_Stream0->PAR  = (uint32_t)src;
    DMA2_Stream0->M0AR = (uint32_t)dst;
    DMA2_Stream0->NDTR = BUF_SIZE;

    // 4. 配置控制暫存器
    DMA2_Stream0->CR = 0;
    DMA2_Stream0->CR |= (2 << 6)   | // Memory-to-Memory
                        DMA_SxCR_MINC | // 目的地遞增
                        DMA_SxCR_PINC | // 來源遞增
                        DMA_SxCR_TCIE;  // 關鍵：開啟「傳輸完成中斷」

    // 5. NVIC 核心中斷設定
    NVIC_SetPriority(DMA2_Stream0_IRQn, 5); // 設定中斷優先級
    NVIC_EnableIRQ(DMA2_Stream0_IRQn);      // 在 NVIC 中打開門戶
}

void DMA2_Stream0_IRQHandler(void) {
    // 檢查是不是「傳輸完成 (TC)」引發的中斷
    if (DMA2->LISR & DMA_LISR_TCIF0) {
        // 【在此處執行搬運完後的任務】
        // 例如：啟動下一個任務、處理數據、或通知系統
        Data_Processing_Complete_Callback(); 
        // 必須手動清除標誌，否則會卡死在裡面
        DMA2->LIFCR |= DMA_LIFCR_CTCIF0;
    }
}

int main(void) {
    DMA_M2M_Init_With_IT();
    DMA2_Stream0->CR |= DMA_SxCR_EN;
    while(1) { __WFI();} // Wait For Interrupt (進入睡眠，直到中斷喚醒)
}
```
__WFI() 是 ARM 架構下的晶片所提供的指令集，有中斷發生時才會喚醒 CPU，否則就是在低功耗模式下。這種模式就需要等待 CPU 把緩衝區的資料搬走才能夠再放進去，所以會有比較多的時間在等待，如果資源夠的話就可以使用雙緩衝模式。

#### 2. 雙緩衝模式 Double Buffer
雙緩衝就是多了一個空間，當滿了時就可以去把資料塞進另一個空間，就可以減少 CPU 等待的時間
```C
#include "stm32f4xx.h"
#define BUF_SIZE 16
uint32_t src_data[BUF_SIZE], dst_buffer0[BUF_SIZE], dst_buffer1[BUF_SIZE];
volatile uint32_t b0_count = 0, b1_count = 0;

void DMA2_M2M_DoubleBuffer_Init(void) {
    // 1. 開啟 DMA2 時鐘 (M2M 必須使用 DMA2)
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;

    // 2. 確保 Stream 停止才能配置
    DMA2_Stream0->CR &= ~DMA_SxCR_EN;
    while(DMA2_Stream0->CR & DMA_SxCR_EN);

    // 3. 配置傳輸地址
    // PAR 在 M2M 模式下充當「來源地址」
    DMA2_Stream0->PAR  = (uint32_t)src_data;
    // 雙緩衝需要設定兩個記憶體地址 (M0AR 與 M1AR)
    DMA2_Stream0->M0AR = (uint32_t)dst_buffer0;
    DMA2_Stream0->M1AR = (uint32_t)dst_buffer1;

    // 4. 設定搬運數量 (每個 Buffer 的長度)
    DMA2_Stream0->NDTR = BUF_SIZE;

    // 5. 配置控制暫存器 (CR)
    DMA2_Stream0->CR = 0;      // 清空初始值
    DMA2_Stream0->CR |= (2 << 6);   // DIR: 10 -> Memory-to-memory
    DMA2_Stream0->CR |= (2 << 13);  // MSIZE: 32-bit (Word)
    DMA2_Stream0->CR |= (2 << 11);  // PSIZE: 32-bit (Word)
    DMA2_Stream0->CR |= DMA_SxCR_MINC; // 目的地地址遞增
    DMA2_Stream0->CR |= DMA_SxCR_PINC; // 來源地址遞增 (M2M 時 PINC 代表來源)
    
    // --- 雙緩衝核心配置 ---
    DMA2_Stream0->CR |= DMA_SxCR_DBM;  // 開啟雙緩衝模式
    DMA2_Stream0->CR |= DMA_SxCR_CIRC; // 雙緩衝必須啟用環狀模式
    DMA2_Stream0->CR |= DMA_SxCR_TCIE; // 開啟傳輸完成中斷

    // 6. NVIC 中斷控制器設定
    NVIC_SetPriority(DMA2_Stream0_IRQn, 0);
    NVIC_EnableIRQ(DMA2_Stream0_IRQn);

    // 7. 啟動 DMA
    DMA2_Stream0->CR |= DMA_SxCR_EN;
}
void DMA2_Stream0_IRQHandler(void) {
    // 檢查是否為傳輸完成中斷 (TCIF0)
    if (DMA2->LISR & DMA_LISR_TCIF0) {
        // 判斷硬體「現在正在」搬哪一塊，進而推論「剛搬完」哪一塊
        // CT = 1 代表現在正搬往 Buffer 1，表示 Buffer 0 剛完成
        if (DMA2_Stream0->CR & DMA_SxCR_CT) b0_count++; 
        else b1_count++;
        // 必須清除標誌位
        DMA2->LIFCR |= DMA_LIFCR_CTCIF0;
    }
}

int main(void) {
    for(int i = 0; i < BUF_SIZE; i++) src_data[i] = i;
    DMA2_M2M_DoubleBuffer_Init();
    while(1) {__WFI(); }
}
```
雖然雙緩衝可以加速，但如果 CPU 時脈夠高，還是會有空窗期發生。然而通常在傳遞時資料大小會大過緩衝區大小非常多，所以會使用環形+雙緩衝模式。
#### 3. 環形+雙緩衝模式 Circular+Double Mode
當緩衝區 A 滿了資料就放到緩衝區 B，然後 CPU 去緩衝區 A 搬資料，緩衝區 B 滿了且緩衝區 A 資料也搬完了就在做切換，不同的在這兩塊緩衝區做切換直到搬完資料，這樣就可以在使用少量記憶體的情況下逼近硬體的理論上限。
```C
#include "stm32f4xx.h"

#define BUF_SIZE 16

// 來源資料
uint32_t src_data[BUF_SIZE] = {0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15};

// 目的地雙緩衝
uint32_t dst_buf0[BUF_SIZE];
uint32_t dst_buf1[BUF_SIZE];

volatile uint32_t transfer_cnt = 0;
volatile uint8_t  active_buffer = 0; // 0: 剛搬完 buf0, 1: 剛搬完 buf1

void DMA2_Stream0_DoubleBuffer_Init(void) {
    // 1. 開啟 DMA2 時鐘 (M2M 只能用 DMA2)
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;

    // 2. 停止 Stream 並等待硬體就緒
    DMA2_Stream0->CR &= ~DMA_SxCR_EN;
    while(DMA2_Stream0->CR & DMA_SxCR_EN);

    // 3. 設定地址
    // PAR: 在 M2M 模式下作為 Source (來源)
    DMA2_Stream0->PAR  = (uint32_t)src_data;
    // M0AR & M1AR: 雙緩衝的目的地
    DMA2_Stream0->M0AR = (uint32_t)dst_buf0;
    DMA2_Stream0->M1AR = (uint32_t)dst_buf1;

    // 4. 設定搬運數量
    DMA2_Stream0->NDTR = BUF_SIZE;

    // 5. 配置控制暫存器 CR
    DMA2_Stream0->CR = 0;
    DMA2_Stream0->CR |= (2 << 6);   // DIR: 10 -> Memory-to-Memory
    DMA2_Stream0->CR |= (2 << 13) | (2 << 11); // MSIZE & PSIZE: 32-bit
    DMA2_Stream0->CR |= DMA_SxCR_MINC | DMA_SxCR_PINC; // 兩端地址皆遞增

    // --- 極致效能核心設定 ---
    DMA2_Stream0->CR |= DMA_SxCR_DBM;  // 開啟雙緩衝模式
    DMA2_Stream0->CR |= DMA_SxCR_CIRC; // 雙緩衝必須配合環狀模式
    DMA2_Stream0->CR |= DMA_SxCR_TCIE; // 開啟傳輸完成中斷

    // 6. NVIC 配置
    NVIC_SetPriority(DMA2_Stream0_IRQn, 0);
    NVIC_EnableIRQ(DMA2_Stream0_IRQn);

    // 7. 啟動 DMA
    DMA2_Stream0->CR |= DMA_SxCR_EN;
}

// --- 中斷服務程式 ---
void DMA2_Stream0_IRQHandler(void) {
    // 檢查傳輸完成標誌 (TCIF0)
    if (DMA2->LISR & DMA_LISR_TCIF0) {
        // 當中斷發生時，硬體已經「翻轉」了 CT 位元。
        if (DMA2_Stream0->CR & DMA_SxCR_CT) active_buffer = 0; 
        else active_buffer = 1;
        transfer_cnt++;
        // 必須清除標誌
        DMA2->LIFCR |= DMA_LIFCR_CTCIF0;
    }
}

int main(void) {
    DMA2_Stream0_DoubleBuffer_Init();
    while(1) { __WFI(); }
}
```
在搬運的過程中如果 CPU 太快則還是需要等待，所以如果資源夠的話可以去計算 CPU 搬運時間跟塞滿緩衝區的時間來決定要幾個緩衝區。
