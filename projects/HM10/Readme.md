STM32F429 Bluetooth (HC-05/06) 通訊專案
本專案實作了 STM32F429I-Discovery 開發板透過 USART3 與藍牙模組（如 HC-05 或 HC-06）進行非同步通訊的功能。專案演示了如何使用「中斷接收」機制來處理來自手機端（或另一個藍牙設備）的指令，並實現數據的回傳（Echo）與狀態指示。

## 一、功能特點
1. 中斷式接收 (Interrupt-Driven RX)：使用 HAL_UART_Receive_IT 實作非阻塞式接收，確保系統在等待數據時仍能執行其他任務。
2. 字元回傳機制 (Echo Back)：當開發板收到來自藍牙的數據時，會立即透過 HAL_UART_Transmit 將相同字元傳回手機端，方便在 Serial Terminal 上驗證連線。
3. 緩衝區管理：實作了簡易的 rx_buffer 陣列與索引管理，並能識別換行符號（\n 或 \r）作為指令結尾。
4. 接收指示：每當成功接收到一個位元組時，開發板上的 LED2 (GPIOG Pin 14) 會翻轉狀態（閃爍）。

## 一、硬體連接
1. 開發板：STM32F429I-DISCO
2. 藍牙模組：HC-05 / HC-06 / JDY-31 等支援 UART 協議的模組。
3. 引腳配置 (USART3)：\
STM32 PB10 (TX) → 藍牙 RX\
STM32 PB11 (RX) → 藍牙 TX\
VCC → 3.3V\
GND → GND

## 三、軟體設定
1. UART 參數：\
Baud Rate: 9600\
Word Length: 8 Bits\
Stop Bit: 1 Bit\
Parity: None
2. 關鍵回呼函式：在 HAL_UART_RxCpltCallback 中處理數據存儲與邏輯判斷，並在結束後重新啟動 HAL_UART_Receive_IT 以維持持續接收。

## 三、快速上手
1. 環境：使用 STM32CubeIDE 開啟專案。
2. 接線：按照「硬體連接」部分連接藍牙模組與開發板。
3. 配對：使用手機下載 "Bluetooth Serial Terminal" 等 APP，並與藍牙模組配對（通常預設密碼為 1234 或 0000）。
4. 測試：燒錄程式後，可利用手機或是串口助手先配對，從一端發送隨意字元。就會看到回傳的字元。開發板上的紅色 LED 會隨著數據接收而閃爍。\
https://serial.baud-dance.com/#/
