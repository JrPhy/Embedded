使用 STM32 硬體來實作，會有介紹通訊協定，如 GPIO、UART、I<sup>2</sup>C、SPI、CANBUS

I<sup>2</sup>C 結構較簡單但傳輸距離較短，也容易受到電磁干擾，傳輸速率由主從設備規定鮑率決定，無法同時收發訊號。\
SPI 主設備會與每個從設備單獨連線，結構較複雜但傳輸速率較快，主設備可以同時收發訊號。\
CANBUS 沒有主從關係，為廣播模式。

複雜度：SPI > CANBUS > I<sup>2</sup>C\
傳輸速率：SPI > CANBUS > I<sup>2</sup>C\
傳輸距離：CANBUS > SPI > I<sup>2</sup>C

使用 STM32F429I-DISC1 和 CubeIDE 開發，專案可看 .zip 檔
![img](Board_selector.png)

![img](New_project.png)
