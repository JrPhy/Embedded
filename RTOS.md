在嵌入式開發中，單晶片開發稱為 bare metal (裸機)，需要自己去控制 CPU 與記憶體，如果只有少量感測器的數據要讀取就會使用這種模式。而當感測器一多起來，雖然仍可以自己管理，但是交給 OS 對開發者來說是比較好的，但畢竟不是每個晶片都有很強的 CPU 與 MMU，所以在一些手錶上面就會使用即時操作系統 RTOS。不同於一般的 OS，RTOS 要求的是即時性與確定性，也就是某個任務要在一定時間內完成，現今流行的 RTOS 有以下幾個

| RTOS | 開源性質 | 記憶體占用 | 主要優點 |
| --- | :---: | :---: | :---: |
| FreeRTOS | MIT | 約 4-9 KB | 移植性最強 |
| Zephyr | Apache | 中偏小 | 通訊協定多 |
| ThreadX | MIT | 約 2 KB | 安全認證齊全 |
| VxWorks | 閉源商用 | 較大 | 工業標竿 |

在此已 FREERTOS 為主，因為業界使用較多且主要的 API 都已經寫好，只要去呼叫即可。裡面的文件與用途有以下
```
port.c : 針對不同硬件平台的接口
heap_4.c : 內存管理相關
croutine.c : 協程相關
event_groups.c : 事件標志組相關
list.c : 列表，FreeRTOS的一種基礎數據結構
queue.c : 隊列相關
tasks.c : 任務創建、掛起、恢覆、調度相關
timers.c : 軟件定時器相關
```

## 一、任務
相較於一般的 OS，RTOS 通常是跑單核心，以任務(Task)為單位執行，可以看成只在上面執行一個主程式，裡面的任務看作是這程式的執行緒，所有 Task 共享同一個位址空間，所以在 [Context Switch](https://github.com/JrPhy/Multiple_Thread/blob/main/%E4%B8%8A%E4%B8%8B%E6%96%87%E4%BA%A4%E6%8F%9B%E8%88%87%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C.md) 也會更輕量更快。狀態有以下幾種
![IMG](https://freertos.org/media/2018/tskstate.gif)
[SOURCE](https://freertos.org/zh-cn-cmn-s/Documentation/02-Kernel/02-Kernel-features/01-Tasks-and-co-routines/02-Task-states)
#### 1. 任務創建
如同單晶片上的主程式，每個任務裏面也會有一個無窮迴圈，主要邏輯放在無窮回圈內並放 osDelay、osDelayUntil、Semaphore 或 Queue 來讓低優先級的任務執行，初始化的邏輯放在無窮迴圈外面，
```C
void MyTask(void *argument) {
    // 初始化邏輯
    for(;;) {
        // 主要邏輯
        /* 3. 延遲 (必備！否則低優先權任務會餓死) */
        osDelay(10); 
    }
}
```
如果是最低優先權的任務也要有 osDelay(0)，這樣才會主動讓出 CPU，且任務被刪除時才會回收記憶體。在原生的 FreeRTOS 中使用 xTaskCreate 來建立，而 CUBEIDE 中的 FreeRTOS 則有另一個封裝，是 ARM 推出的通用標準 (CMSIS)，這邊也用後者，其寫法樣板如下
```C
#include "cmsis_os.h"
...
const osThreadAttr_t attributes = {
  .name = "MyTask",
  .stack_size = 1024,   // Stack 大小 (Bytes)
  .priority = (osPriority_t) osPriorityNormal, 
};

osThreadId_t id = osThreadNew(MyTask, NULL, &attributes);
```
```osThreadId_t osThreadNew (osThreadFunc_t func, void *argument, const osThreadAttr_t *attr)```
第一個就是把任務傳入，第二個則是要傳入該任務的變數，沒有就放 ```NULL```，第三個則是放任務的資訊，如名稱、需要的記憶體與優先及，就是對應到 xCreate 的 API。而在 main 函數中會有
```c
int main(void) {
    // CUBEIDE 會自動生成
    HAL_Init();
    // 初始化 RTOS
    osKernelInitialize();
    // 定義任務名稱、優先級、實例數、堆疊大小
    osThreadDef(uartTask, StartUartTask, osPriorityNormal, 0, 128);
    uartTaskHandle = osThreadCreate(osThread(uartTask), NULL);
    // 啟動內核調度器 (啟動後不再返回)
    osKernelStart();
    // 正常不會到這裡
    while (1) {}
}
```
這樣就完成了 RTOS 的任務創建。

#### 2. 任務切換
Process、Thread 與 Task 在切換時都會需要存下 Control Block 的資訊，只是裡面所需的內容不同而已，Task 因為都在同一塊記憶體中所以會更輕量。FreeRTOS TCB 存儲了以下成員
```
pxTopOfStack：指向任務棧頂，這裡存著 R0-R15、xPSR。
xStateListItem：讓排程器知道這個任務是在 Ready 還是 Blocked 隊列。
uxPriority：決定誰能「搶佔」CPU 的關鍵。
pxStack：指針，用來偵測 Stack Overflow。
```
當然 FreeRTOS 在底層都已經做掉了，只需在 CUBEIDE 中做以下設定
```
System Core -> NVIC:
Time base: System tick timer 的優先級通常設為 15 (最低)。
PendSV interrupt 的優先級也必須是 15 (最低)。

Middleware -> FREERTOS:
確保 USE_PREEMPTION (開啟搶佔) 設為 Enabled。
```
如果遇到相同優先序的任務，則會每隔一小段時間就切換到另個任務，會使得 CPU 資源耗費許多在任務切換上，所以會盡量把優先及分開，且越高的 osDelay 時間要越短。不過若要避免這種情況發生，例如兩任務有優先順序時就可以用 Queue 隊列。

## 二、[隊列 (Queue) ](https://github.com/JrPhy/DS-AL/blob/master/Stack_and_Queue/Queue-%E4%BD%87%E5%88%97.md)
為一種先進先出的資料結構，FreeRTOS 中隊列與信號都是**事件**，也就是有接收到訊號時才會去執行該任務，而 osDelay 是輪詢，在固定的時間去執行，若要傳資料且資料會一筆一筆處理、或是兩個任務有順序性，如收集數據在處理數據就可以用 Queue。
```C
#include "main.h"
#include "cmsis_os.h"

osMessageQDef(sensorQueue, 8, uint32_t);
osMessageQId sensorQueueHandle;

osThreadId sensorTaskHandle;
osThreadId controlTaskHandle;

void SensorTask(void const *argument);
void ControlTask(void const *argument);

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    /* 建立 Queue（這裡才真的產生 Queue） */
    sensorQueueHandle = osMessageCreate(
        osMessageQ(sensorQueue),
        NULL
    );

    /* 建立 Sensor Task */
    osThreadDef(sensorTask, SensorTask, osPriorityNormal, 0, 128);
    sensorTaskHandle = osThreadCreate(osThread(sensorTask), NULL);

    /* 建立 Control Task */
    osThreadDef(controlTask, ControlTask, osPriorityAboveNormal, 0, 128);
    controlTaskHandle = osThreadCreate(osThread(controlTask), NULL);

    osKernelStart();

    /* 理論上不會跑到這裡 */
    while (1) {}
}

void SensorTask(void const *argument) {
    uint32_t temperature;
    for (;;) {
        temperature = Read_Sensor();  // 例如 ADC/I2C
        osMessagePut(sensorQueueHandle, temperature, 0);
        osDelay(100);
    }
}

void ControlTask(void const *argument) {
    osEvent evt;
    uint32_t temp;

    for (;;) {
        /* 等待 Queue 裡有資料 */
        evt = osMessageGet(sensorQueueHandle, osWaitForever);
        if (evt.status == osEventMessage) {
            temp = evt.value.v;
            if (temp > 50) Turn_On_Fan();
            else Turn_Off_Fan();
        }
    }
}
```
在最一開始的 ```osMessageQDef(sensorQueue, 8, uint32_t);``` 定義了隊列的資料結構，sensorQueue 可以當成這個隊列的名稱，大小為 8，也就是最多只有 8 個任務可以等待，超過就看是要阻塞還是要放棄，要傳遞的資料是 uint32_t，剩下的步驟就類似創建任務。在任務函數中，```osMessagePut``` 就是把資料放入，```osMessageGet``` 則是把資料取出。

## 三、多條件同步
在有些時候我們會希望整個系統與傳感器都連接上後再開始執行某個任務，就可以使用 ```EventFlags``` 來等待多個事件達到條件後再往執行。在上述例子中加入下方的程式碼後就可以得到通知。
```C
#include "cmsis_os2.h"
#define EVT_INIT_OK     (1U << 0)
#define EVT_INIT_FAIL   (1U << 1)
osEventFlagsId_t systemEvents;
// ...
int main() {
    // ...
    systemEvents = osEventFlagsNew(NULL);
    // ...
    while (1) {}
}

void SensorTask(void *argument) {
    // 初始化完成後通知系統
    osEventFlagsSet(systemEvents, EVT_SENSOR_READY);
    // ...
}

void ControlTask(void *argument) {
    // 等待條件成立
    osEventFlagsWait(
        systemEvents,
        EVT_SENSOR_READY | EVT_SYSTEM_READY,  // 兩個條件
        osFlagsWaitAll,                       // AND（全部要成立）
        osWaitForever
    );
    // ...
}
```
當然有多個感測器就會有更複雜的寫法，所以就可以用 Event 來告訴我們哪些成功哪些失敗。

## 四、[信號量 (Semaphore) ](https://github.com/JrPhy/Multiple_Thread/blob/main/%E7%AB%B6%E7%88%AD%E6%A2%9D%E4%BB%B6%E8%88%87%E9%8E%96.md#3-%E8%99%9F%E8%AA%8C-semaphore)
可以看成長度為 1 的隊列，不過傳入的參數也可以 > 1。常用來等通知或喚醒設備，例如一個儀器長時間不用進入待機，當使用者按下按鈕或是有動作時要能快速喚醒。
```C
#include "cmsis_os2.h"

void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_I2C1_Init(void);
void Enter_Stop_Mode(void);

osThreadId_t watchTaskHandle;
osSemaphoreId_t accelSem;

void WatchTask(void *argument);

int main(void) {
    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_I2C1_Init();

    osKernelInitialize();

    const osSemaphoreAttr_t accelSem_attr = {
        .name = "AccelWakeSem"
    };
    accelSem = osSemaphoreNew(1, 0, &accelSem_attr);
    if (!accelSem) Error_Handler();

    /* ===== Create Watch Task ===== */
    const osThreadAttr_t watchTask_attr = {
        .name = "WatchTask",
        .priority = osPriorityNormal,
        .stack_size = 1024
    };
    watchTaskHandle = osThreadNew(WatchTask, NULL, &watchTask_attr);
    if (!watchTaskHandle) Error_Handler();

    osKernelStart();

    while (1) {}
}
void MX_FREERTOS_Init(void) {
    const osSemaphoreAttr_t accelSem_attr = {
        .name = "AccelWakeSem"
    };

    // Binary semaphore：最大 1，初始 0（一開始鎖住）
    accelSem = osSemaphoreNew(1, 0, &accelSem_attr);
    if (!accelSem) Error_Handler();}

    // 建立手錶 Task
    osThreadNew(WatchTask, NULL, &watchTask_attr);
}

void WatchTask(void *argument) {
    for (;;) {
        Enter_Stop_Mode();
        /* ===== 等待加速度計喚醒 ===== */
        if (osSemaphoreAcquire(accelSem, osWaitForever) == osOK) {
            Accel_Clear_INT(); // 清加速度計中斷（一定要）
            OLED_ON();
            Update_Time();
            osDelay(5000);
            OLED_OFF();
        }
    }
}

void Enter_Stop_Mode(void) {
    HAL_SuspendTick();
    __HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU);

    HAL_PWR_EnterSTOPMode(
        PWR_LOWPOWERREGULATOR_ON,
        PWR_STOPENTRY_WFI
    );

    SystemClock_Config();   // STM32 必做
    HAL_ResumeTick();
}

void Accel_Clear_INT(void) {
    uint8_t src;
    I2C_ReadReg(LIS3DH_INT1_SRC, &src, 1);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_1) {
        osSemaphoreRelease(accelSem);
    }
}
```
其中的 ```osSemaphoreId_t osSemaphoreNew (uint32_t max_count, uint32_t initial_count, const osSemaphoreAttr_t *attr)``` 第一個參數就是信號數，用 1 時通常代表開關或是一個事件有沒有發生，> 1 則表示還有多少資源可用，例如資源池等。

| | Semaphore | Qeueu | EventFlags  |
| --- | :---: | :---: | :---: |
| 適合 | 不用給資料 | 要傳資料 | 不需要資料，只要知道來源 |
| 適合 | 單一事件發生過了嗎 | 事件有順序 | 多個事件哪個發生 |
| 適合 | 還有幾個資源能用 | 每次發生都要被處理 |  |

要使用哪個可用以下準則判斷
```
資料 → Queue
來源 → EventFlags
數量 → Semaphore
```
