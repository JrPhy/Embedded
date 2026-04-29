STM32F429 FreeRTOS Gesture UI 專案
本專案在 STM32F429I-Discovery 開發板上運行 FreeRTOS 即時作業系統，並透過內建的 LCD 顯示器 與 觸控面板 (TS) 實作了一套手勢切換分頁的 UI 系統。使用者可以透過左右滑動手勢，在不同的資訊頁面（如空氣品質、天氣資訊）之間切換。

## 一、功能特點
1. 多任務並行處理 (RTOS)：
2. Touch Task：負責監測觸控面板狀態，計算手指位移並判定「左滑」或「右滑」手勢。
3. UI Task：作為消費者任務，從訊息隊列 (Message Queue) 接收手勢指令並更新分頁顯示。
4. 手勢識別演算法：實作於 gesture_ui.c，透過計算觸控座標的 ΔX 位移並設定門檻值（Threshold），精確判定滑動方向。
5. 中斷與通訊機制：使用 FreeRTOS 的 osMessageQueue 進行任務間通訊，確保 UI 更新的流暢度與解耦。
6. 記憶體優化：利用外部 SDRAM 作為 LCD 的 Frame Buffer (顯存)，支援 240x320 解析度的畫面渲染。

## 二、硬體資源
1. 開發板：STM32F429I-DISCO (內建 LTDC 控制器、STMPE811 觸控晶片、外部 SDRAM)。
2. 螢幕設定：解析度 240x320，使用 Layer 0 進行顯示，底色預設為黑色。
3. 手勢定義：\
向左滑動：切換至下一頁 (currentPage++)。\
向右滑動：切換至前一頁 (currentPage--)。

## 三、軟體架構
1. main.c：系統核心初始化，包含時鐘 (180MHz)、SDRAM、LTDC 及 FreeRTOS 內核啟動。
2. gesture_ui.c / .h：UI 系統核心模組。
3. UI_System_Init()：初始化 LCD、觸控晶片與建立 Message Queue。
4. StartTouchTask()：生產者任務，負責手勢偵測。
5. StartUITask()：消費者任務，負責分頁渲染邏輯。
6. freertos.c：由 STM32CubeMX 生成的 RTOS 任務配置檔。
7. FreeRTOSConfig.h：FreeRTOS 核心參數設定（如 Tick rate, Heap size 等）。
8. freeRTOS.ioc：STM32CubeMX 專案設定檔。

## 四、快速上手
1. 環境需求：使用 STM32CubeIDE 開啟此專案。
2. 編譯與燒錄：編譯專案並將韌體燒錄至 STM32F429I-Discovery 開發板。

## 五、操作說明：
開機後螢幕會顯示預設分頁（PAGE_AIR）。在觸控螢幕上向左或向右快速滑動，螢幕將顯示 "gesture" 並切換至對應分頁。當手指按壓螢幕時，下方會顯示 "TouchDetected" 字樣作為即時反饋。

## 六、注意事項
1. SDRAM 依賴：本專案必須正確初始化外部 SDRAM 才能正常顯示畫面，顯存起始位址設為 0xD0000000。
2. 手勢門檻：目前程式中定義向右滑動門檻為 +20 像素，向左滑動為 −30 像素，可根據使用習慣在 gesture_ui.c 中修改。
