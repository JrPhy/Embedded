## 一、MCU/SOC/PC 的開機
當按下硬體的電源鍵後會並不會進入系統，而是會先初始化一些設備，開機載入程式 (Bootloader) 通常放在非揮發性記憶體的固定起始位址，隨著不同的 MCU 或 SOC 而有不同，例如 ARM 為0x000000004， MIPS為0xBFC00000，可以在各廠家的使用手冊上找到，這樣 CPU 在上電或重置時就能直接執行 Bootloader 的初始化程式。當然起始位置會。Bootloader 會去初始化一些外設設備，如時鐘系統 (Clock)、GPIO、UART、SPI、I2C 等必要接口，甚至還有 WIFI 跟 BLE，就可以拿來做更新。也會去建立記憶體映射表，也就是虛擬位置到實際地址的表，再去跳轉到主程式的部分。而在 PC 上則是會先進到 BIOS/UEFI，裡面會有許多 Bootloader 裝在 MBR 上，且會有預設的 OS，也可安裝多個 OS 讓使用者選擇。所以 Bootloader 其實是一個非常小的程式，放在開機後記憶體的起始位置，初始化一些設備並建立記憶體映射表後再跳轉進到主程式。

## 二、Bootloader
在 MCU 或 SOC 開機後會直接進到 bootloader，現在較常見的有 U-boot 與 Redboot，主要會有兩個階段，這邊已 UBOOT 為例．第一階段為初始化 CPU / stack / SRAM / clock 等基本硬體，由組語寫成，通常越小越好．也幾乎僅可用 SRAM 如 flash/NVRAM 等，有些也在 ROM 內．第二階段則是由組語跳轉到 main() 函數，之後則都用 C 寫成．是用來引導啟動 kernel，最终目的就是從 flash 中讀取 kernel 放到 RAM 中並啟動．在 [U-Boot](https://github.com/u-boot/u-boot) 裡，```u-boot.lds(u-boot/arch/arm/cpu/u-boot.lds)``` 是一個鏈接腳本 (Linker Script)，它的作用是告訴編譯器/鏈接器程式的各個段 (section) 要放到記憶體的哪個位置，不同架構的 MCU/SOC 都有自己的 u-boot.lds。有關 u-Boot 詳細步驟解說可看[這篇文章](https://zhuanlan.zhihu.com/p/659724837)，大略流程可參考[這篇文章](https://www.cnblogs.com/unclemac/p/12783412.html)及下圖\
![img](https://img-blog.csdnimg.cn/20190204163123270.png)

## 三、U‑Boot 的源碼目錄結構
| 目錄名稱       | 作用說明                                                                 |
|----------------|--------------------------------------------------------------------------|
| arch/          | 不同 CPU 架構的程式碼 (ARM, RISC-V, x86, MIPS...)，包含啟動入口、低階初始化、SPL 支援 |
| board/         | 板級相關程式碼，每個廠商/開發板獨立目錄，放置 DRAM 初始化、時鐘設定、SPL 程式等       |
| configs/       | 各開發板的 defconfig 檔案，定義編譯選項 (是否啟用 SPL、支援哪些驅動)                  |
| include/       | 標頭檔目錄，包含全域設定 (config.h)、API 定義、資料結構                             |
| common/        | 架構無關的通用函式，例如環境變數處理、命令解析                                     |
| cmd/           | U‑Boot 命令的實作，例如 bootm、help、printenv                                 |
| drivers/       | 各類硬體驅動程式：UART、MMC、NAND、SPI、USB、網路等                             |
| boot/          | 與映像載入、啟動相關的支援程式                                                 |
| dts/           | Device Tree 檔案，描述硬體資源                                                 |
| lib/           | 通用函式庫，例如壓縮、CRC、字串處理                                           |
| net/           | 網路協定棧與相關功能                                                         |
| fs/            | 檔案系統支援，例如 FAT、EXT4                                                 |
| tools/         | 編譯時工具，例如 mkimage 生成 U‑Boot 可識別的映像                               |
| doc/           | 官方文件，說明移植、配置、編譯方法                                           |
| test/          | 測試程式與框架                                                               |

移植時會需要改以下檔案
| 目錄名稱       | 作用說明                                                                 |
|----------------|--------------------------------------------------------------------------|
| 必改         | board/ (板級初始化)、configs/ (編譯設定)、arch/ (架構相關啟動程式) |
| 必備         | dts/ (描述硬體)、include/ (設定檔) |
| 選用         | drivers/ (依需求加入驅動) |

其中的 board 資料夾為一些廠家提供的 uboot，詳細可參考[這篇文章](https://blog.csdn.net/qq_52479948/article/details/128900841)。

## 四、第一階段 SPL
BootROM (SoC 內建) 上電後 CPU 從 Reset Vector 開始執行，通常是晶片廠商提供的 BootROM。它的任務是把 SPL 載入到片上 SRAM。

SPL 程式碼就在 U‑Boot 原始碼裡的 spl/ 框架，以及 arch/ 和 board/ 目錄下的 SPL 專用程式。會去初始化最基本的硬體 (時鐘、PLL、堆疊)，然後從啟動裝置 (NAND、SD、SPI Flash 等) 載入完整的 U‑Boot proper，再跳轉到第二階段入口 (_start)。

## 五、第二階段
#### 1. _start
u-boot.lds 中的 ENTRY(_start) 就是 U-boot 的進入點，其中 ARMv7 _start 放在 ```u-boot/arch/arm/lib/vectors.S``` 或是在每個 ```cpu/start.S``` 中，ARMv8 則是統一在 start.S 中，裡面可看到一開始先做了 reset，之後就到 ```.globl _start```，裡面的 ```.section ".vectors", "ax"```．\
.section: 將接下來的程式碼或資料放入後面得變數中\
".vectors": 變數名稱\
"ax": 可分配 allocatable 與可執行 executable\
所以意思就是將接下來的程式碼放入 .vectors 中，並標注可分配與可執行．後面就是放要放進去的程式碼與變數了。\
進入 _start 後會先關閉中斷並初始化 stack，清零 BSS 段、設定向量表並準備全域資料，之後就跳轉到 C 語言。ARMv8 還會去設定 MMU。

#### 2. board_init_f()
在進入 main 之前還需要先初始化一些基本硬體，如 DRAM、UART、clock 等，然後會把 U-Boot 搬到 DRAM，並建立更完整的 stack，然後在跳往下一階段

#### 3. board_init_r()
這步會初始化更上層的硬體如 I/O、storage、network，然後載入環境變數並準備命令列嶼啟動流程。

#### 4. main_loop()
這步就是真正的 main 函數了，會去處理 boot script 或啟動 kernel

### 六、移植
