## 一、SPI
全名為 Serial Peripheral Interface。SPI 常用在 ADC, DAC, SRAM 等，是一種同步，全雙工的通訊協定。走線上比 UART 多了一根時鐘線，用來同步接收端的時間訊號，也因為有了時間訊號，所以傳輸時不再需要傳起始位與停止位，所以傳輸速率會比 UART 與 I<sup>2</sup>C 快，但在結構上就多了時間線。若為一對多則是用 CS 線來選定要發送的設備，且 CS 線只會與唯一一個設備被連接。發送時間訊號的為 Master，接收時間訊號的為 Slave。因為訊號是由 Master 傳給 Slave，故 Master 的 Tx 稱為 MOSI(Master Out Slave In)，此時 Slave 的 Rx 端稱為 SDI/SI(Slave (Data) In)，反過來則為 MISO 與 SDO。

| Slave 2 | SPI | Master | SPI | Slave 1 |
| --- | --- | --- | --- | --- |
| CLK | <-- | CLK | --> | CLK |
| SDI | <-- | MOSI | --> | SDI |
| SDO | --> | MISO | <-- | SDO |
|   |  | CS 1 | --> | CS 1 |
| CS 2 | <-- | CS 2 |  | CS 1 |

![https://zh.wikipedia.org/wiki/%E5%BA%8F%E5%88%97%E5%91%A8%E9%82%8A%E4%BB%8B%E9%9D%A2#/media/File:SPI_three_slaves.svg](https://upload.wikimedia.org/wikipedia/commons/f/fc/SPI_three_slaves.svg)

#### 1. 傳輸協定
SPI 主要是靠時鐘訊號來做讀取，也可以暫停再開始。要發送資料時，會先將 CS 的電位改變，要拉高還是拉低完全由晶片手冊決定，在此以拉低為例。此時時中線會一直發時鐘訊號，再時鐘線翻轉時，也就是上升沿或下降沿時的讀取，在此以上升沿為例。資料線也會一直發送訊號出去，所以讀取訊號時的電路如下圖
![SPI 讀取電路時的電位](https://github.com/JrPhy/Firmware/blob/main/pic/SPI.jpg) https://en.wikipedia.org/wiki/Serial_Peripheral_Interface \
總共會有四種模式

| MODE | CS | CLK | MOSI | Slave 1 |
| --- | --- | --- | --- | --- |
| 0 | 低電位 | 上升沿 | --> | CLK |
| 1 | 低電位 | 下降沿 | --> | SDI |
| 2 | 高電位 | 上升沿 | <-- | SDO |
| 3 | 高電位 | 下降沿 | --> | CS 1 |

若是有多個從設備，則會先將其中一個 CS 線訊號拉低，其餘保持高阻抗，結束後再向下一個傳遞訊號。

#### 2. 多個從設備
SPI 雖然與從設備有獨立的時鐘線，但是 TX/RX 仍只有一組，所以無法同時與多個從設備通信，而且每個從設備的時鐘線頻率也不同，所以與不同的從設備通信時要再去做時鐘頻率的初始化
```C
void SPI_Switch_Settings(uint32_t prescaler, uint32_t cpol, uint32_t cpha) {
    // 1. 檢查 SPI 是否正在忙碌
    while (HAL_SPI_GetState(&hspi1) != HAL_READY);

    // 2. 修改參數
    hspi1.Init.BaudRatePrescaler = prescaler;
    hspi1.Init.CLKPolarity = cpol;
    hspi1.Init.CLKPhase = cpha;

    // 3. 重新初始化（HAL_SPI_Init 會處理暫存器的解鎖與重新寫入）
    if (HAL_SPI_Init(&hspi1) != HAL_OK) {
        Error_Handler();
    }
}

// 假設 APB 總線為 84MHz，分頻 64 = 1.3MHz
SPI_Switch_Settings(SPI_BAUDRATEPRESCALER_64, SPI_POLARITY_LOW, SPI_PHASE_1EDGE);

HAL_GPIO_WritePin(GPIOA, CS_DEV_A_Pin, GPIO_PIN_RESET);
HAL_SPI_Transmit(&hspi1, dataA, 1, 10);
HAL_GPIO_WritePin(GPIOA, CS_DEV_A_Pin, GPIO_PIN_SET);

// 分頻 4 = 21MHz
SPI_Switch_Settings(SPI_BAUDRATEPRESCALER_4, SPI_POLARITY_HIGH, SPI_PHASE_2EDGE);

HAL_GPIO_WritePin(GPIOA, CS_DEV_B_Pin, GPIO_PIN_RESET);
HAL_SPI_Transmit(&hspi1, dataB, 512, 100); // 大數據傳輸
HAL_GPIO_WritePin(GPIOA, CS_DEV_B_Pin, GPIO_PIN_SET);
```

## 二、I<sup>2</sup>C
I<sup>2</sup>C 在電路上是將主設備與從設備並聯，而非拉專用的線去與從設備連接，所以 I<sup>2</sup>C 在電路上會比 SPI 簡潔很多。
![image](https://upload.wikimedia.org/wikipedia/commons/thumb/3/3e/I2C.svg/1920px-I2C.svg.png) ([WIKI](https://zh.wikipedia.org/zh-tw/I%C2%B2C)) \
因為是同時與多個訊號連接，所以在發訊號時會先從主設備發送名稱給從設備，對應的從設備收到後會回應。雖然 I<sup>2</sup> 也可以用同一根線收發訊號，但是沒有辦法同時收發，稱為**半雙工**模式，且因為每個設備的時間精度不同，所以是主從設備訂一個頻率，雙方根據此頻率去讀取訊號，稱為**同步**模式。所以在傳輸訊號上就沒有 SPI 快。

#### 1. 傳輸協定
I<sup>2</sup>C 電路上一條是時間線另一條是數據線，待命時兩條線都是在高電位。當數據線要開始收發數據時，會先將數據線的電為拉低，接著再根據定義的時間頻率決定發出的訊號是多少，且只有在時間線為高電位時，數據線的電位才有意義。收/發完後會告訴設備通信結束，此時會將數據線拉高。也因為需要拉低，所以多數的 I2C 設備都需要去外接一個上拉電阻來拉低訊號。
![image](https://i0.wp.com/www.strongpilab.com/wp-content/uploads/2016/03/i2c_bus_data.png) (https://www.strongpilab.com/i2c-introduction/) \

#### 2. 資料格式
首先會先發送 7 bits 給所有設備，並有 1 bit 的資料說明設備是否有回應，0 無 1 有。接著是每 8 bits = 1 byte 發出去，並帶有 1 bit 的資料說明是否成功，0 無 1 有，最後在發送一個停止位代表結束，所以共為 (7+1)+(8+1)*n+1。\
![image](https://edit.wpgdadawant.com/uploads/news_file/blog/2021/4387/tinymce/_______2021-06-15_______6_24_02.png) (https://www.wpgdadatong.com/blog/detail/44387)

#### 3. 多個從設備
不同於 SPI，I2C 只有共用的時鐘線，所以不需要去重新初始化，只需要傳遞從設備地址即可。
