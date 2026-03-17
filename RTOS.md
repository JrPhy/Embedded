在嵌入式開發中，單晶片開發稱為 bare metal (裸機)，需要自己去控制 CPU 與記憶體，如果只有少量感測器的數據要讀取就會使用這種模式。而當感測器一多起來，雖然仍可以自己管理，但是交給 OS 對開發者來說是比較好的，但畢竟不是每個晶片都有很強的 CPU 與 MMU，所以在一些手錶上面就會使用即時操作系統 RTOS。不同於一般的 OS，RTOS 要求的是即時性與確定性，也就是某個任務要在一定時間內完成，現今流行的 RTOS 有以下幾個

| RTOS | 開源性質 | 記憶體占用 | 主要優點 |
| --- | :---: | :---: | :---: |
| FreeRTOS | MIT | 約 4-9 KB | 移植性最強 |
| Zephyr | Apache | 中偏小 | 通訊協定多 |
| ThreadX | MIT | 約 2 KB | 安全認證齊全 |
| VxWorks | 閉源商用 | 較大 | 工業標竿 |

## 一、與一般 OS 不同處
相較於一般的 OS，RTOS 通常是跑單核心，以任務(Task)為單位執行，所有 Task 共享同一個位址空間，所以在 [Context Switch](https://github.com/JrPhy/Multiple_Thread/blob/main/%E4%B8%8A%E4%B8%8B%E6%96%87%E4%BA%A4%E6%8F%9B%E8%88%87%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C.md) 也會更輕量更快。
