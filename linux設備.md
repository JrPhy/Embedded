硬體的開發會先選擇一個 SOC，然後針對此 SOC 去開發外設設備。隨著 SOC 與外設設備越來越多，開發起來也越來越複雜，所以引入了設備樹的資料結構，開發起來就可以像 C/C++ 一樣去引入檔案就好。在 uboot 中就放了[許多 SOC 的設備樹](https://github.com/u-boot/u-boot/tree/master/dts/upstream/src/arm64/arm)，選擇所使用的 SOC 與平台，然後再去改平台的 .DTS 檔，並引入該平台的 .DSTI 檔。

新增設備要改哪個檔案則是根據如何分層決定，以 foundation-v8 為例，

## 一、dsti
描述 SoC 的核心硬體結構，例如 CPU、記憶體、匯流排、內建控制器等。它屬於「SoC 或平台共用」的基礎設定，讓多個不同板型可以共用這些硬體描述。 ```/ {``` 為根結點，往下看會描述這顆晶片內部所有共用的硬體資源如，所有使用這顆 SoC 的開發板都會共用
```
處理器核心（CPU cores）
內建記憶體（SRAM、ROM）
內部匯流排（bus）
內建控制器（如 UART、SPI、I2C、GIC、DMA 等）
電源管理、時脈、重置控制等
```
#### 1. 新增節點
雖然這部分比較固定，但若是客製晶片可能就會需要新增，例如要在裡面新增一個 spi 匯流排
```
spi_label: spi@spi_addr {
    compatible   = "arm,pl022";
    reg          = <0x0 spi_addr 0x0 spi_size>;
    interrupts   = <GIC_SPI spi_irq_num spi_irq_type>;
    clocks       = <spi_clk_source>;
    clock-names  = spi_clk_name;
    status       = "okay";
};
```
status 就是是否啟用此匯流排，詳細可以看這篇文章 https://zhuanlan.zhihu.com/p/114337560
## 二、.dts
除了上述提到的以外都是放在 .dts，例如外接感測器、LED、按鈕、額外的網路晶片、特殊 GPIO、板上跳線設定等。不同板子可能有不同的外接設備、I/O 配置或硬體差異。若要用此平台新增一個 uart 感測器，首先先確認 dtsi 中是否有 uart 相關匯流排並啟用，啟用後就可以在 .dts 加入以下資訊
```
&soc_uart0 {
    status = "okay";
    sensor@0 {
        compatible = "<廠牌>,<型號>";
        // 其他屬性（如 reg、interrupts、gpios、power 等）
        status = "okay";
    };
};
```
可以看到裡面有兩個 dts 檔案，分別是 dsp 與 fvp，前者是實體板子，後者則是給模擬器用的，選擇要在哪跑後就可以用以下指令編譯了。
```
dtc -I dts -O dtb -o morello-fvp.dtb morello-fvp.dts
```
