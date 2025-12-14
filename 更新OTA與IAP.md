雖然底層軟體不常更新，但至少開發時會需要一直刷新做調整，而變成產品後也會需要更新，故障維修時也常常直接退回出廠設置，所以需要有個地方可以永久存放記憶體，不會因為斷電而重置，也要得到一定權限才能刷寫這塊記憶體，像一些 Android 手機再刷機就是先去要這個權限，之後再把要刷的 kernel 送進手機去做更動。當然刷機不一定都會成功，有時候可能電量不足被關機，此時至少要能退回到上一版本，不論是在 OTA 或是用 IAP。

## 一、OTA 流程
OTA 是透過網路/藍芽/ZigBee/NFC 等通訊協定，把檔案傳到機台上然後再檢查檔案簽章與完整度。可以每隔一段固定時間檢查一次。以現在較新的做法是會去分成 A/B 兩區，假設現在在跑 A，則 A 為啟動區 (active slot)，那就把新的韌體寫到 B，並標記 B 為待驗區 (pending slot)，完成後就可以去重啟 bootloader 然後檢查，如果 B 可以正常使用那就把 B 標記為運行區並把 A 標成備份或非啟動區 (backup slot)，否則就回到 A。下次就是寫入 A。在一些手機等記憶體較大的裝置通常是下載一個全新的 FW，而在一些 MCU 這種記憶體較少的可能會只下載更新包，不過因為這種程式設計較為複雜，所以現在非常少用，不然就是用 IAP，也就是回購買店家幫你接線升級。
```
┌─────────────────┬─────────────────┬─────────────────┐
│   Bootloader    │     APP1        │     APP2        │
│     Flash       │     Flash       │     Flash       │
└─────────────────┴─────────────────┴─────────────────┘
```
```
           +---------------------+
           | 運行於 APP_A        |
           +---------------------+
                     |
                     v
           +---------------------+
           | 下載新韌體至 APP_B  |
           +---------------------+
                     |
                     v
           +---------------------+
           | 校驗 APP_B 韌體     |
           +---------------------+
                     |
           +---------+-----------+
           |                     |
          成功                  失敗
           |                     |
           v                     v
+---------------------+    +---------------------+
| 切換執行至 APP_B     |    | 保持 APP_A 運行      |
+---------------------+    +---------------------+
```
## 二、IAP 流程
雖然 IAP 的優勢就是不用分區，但因為現在都是混用設計居多，所以 IAP 也會同 OTA 一樣做分區設計，否則要更新就都需要插線，例如 USB 或是其他 UART/SPI 等，後者就是使用 SD Card 之類的，其餘的步驟都跟 OTA 一樣。

## 三、整包更新流程
而在開機過程中，BOOTLOADER 預設是進入系統的，所以做更新時必須要把 FLAG 設置成更新，在 u-boot 中可以在 ```u-boot/common/main.c``` 加入 flag 去判斷，這是一般釋出給客戶的做法，因為編譯後客戶就無法去改動，這邊用完整更新為例，也就是下載整包印象檔去刷新。
```C++
void main_loop(void) {
    ...
    if (check_update_flag()) {
        run_command("tftpboot 0x80000000 firmware.img", 0);
        run_command("mmc dev 0", 0);
        run_command("mmc erase 0x1000 0x8000", 0);
        run_command("mmc write 0x80000000 0x1000 0x8000", 0);
        run_command("reset", 0);
    }
    bootdelay_process();
    run_command(getenv("bootcmd"), 0);
    // check for update
}
```
run_command 裡的指令就是 UBOOT SHELL 的指令，也可以去改 setenv + saveenv，這是一般開發的做法。在```configs/<board>_defconfig```中加上
```
CONFIG_BOOTCOMMAND="tftpboot 0x80000000 firmware.img; \
# 定義 U‑Boot 開機後要自動執行的指令序列。這會成為環境變數 bootcmd 的預設值。
# 為從 TFTP server 下載檔案 firmware.img，放到 RAM 位址 0x80000000。
                    sf probe 0; \  #偵測 SPI flash（裝置編號 0）
                    sf erase 0x1000 0x8000; \
# 擦除 flash 上從 0x1000 開始、長度 0x8000 的區塊
                    sf write 0x80000000 0x1000 0x8000; \
# 將 RAM 內的映像檔寫入 flash
                    reset"
```
#### 1. 設定分區
這邊採用直接改 C 語言的作法
```C
#define PART_A_OFFSET 0x100000
#define PART_B_OFFSET 0x500000
#define PART_SIZE     0x400000
// 設定分區起始位置與大小
static int get_current_slot(void) {
    char *slot = env_get("boot_slot");
    if (slot && strcmp(slot, "A") == 0) return 0;
    return 1;
}

static void set_boot_slot(int slot) {
    env_set("boot_slot", slot == 0 ? "A" : "B");
    env_save();
}
```
這兩個函數用來取得跟設定分區
```C
int do_ota_update(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[]) {
    ulong addr, size, offset;
    int next_slot;

    if (argc != 3) {
        printf("用法: ota_update <mem_addr> <size>\n");
        return CMD_RET_USAGE;
    }

    addr = simple_strtoul(argv[1], NULL, 16);
    size = simple_strtoul(argv[2], NULL, 16);

    next_slot = !get_current_slot();
    offset = next_slot ? PART_B_OFFSET : PART_A_OFFSET;

    printf("寫入映像到 slot %c (offset 0x%lx, size 0x%lx)\n", next_slot ? 'B' : 'A', offset, size);

    if (flash_erase(offset, PART_SIZE)) {
        printf("擦除分區失敗\n");
        return CMD_RET_FAILURE;
    }
    if (flash_write(addr, offset, size)) {
        printf("寫入映像失敗\n");
        return CMD_RET_FAILURE;
    }
    if (!verify_image(offset, size)) {
        printf("校驗失敗\n");
        return CMD_RET_FAILURE;
    }
    set_boot_slot(next_slot);
    printf("OTA 更新完成，請重啟系統\n");
    return CMD_RET_SUCCESS;
}

U_BOOT_CMD(
    ota_update, 3, 0, do_ota_update,
    "OTA 雙分區升級: ota_update <mem_addr> <size>",
    "mem_addr: 映像在 DRAM 的地址\n"
    "size: 映像大小"
);
```
## 四、僅更新包流程
生成更新包時可以使用 [bsdiff](https://github.com/mendsley/bsdiff) 這套工具，安裝後輸入已下指令就會產生更新包
```
bspatch old-u-boot.bin new-u-boot.bin uboot.diff
```
會去比較新舊 BIN 檔然後生成 uboot.diff 這個更新包，然後把更新包傳到 device 上，接著把目前啟動分區的寫成一個映像檔，然後用以下指令把更新檔與舊映像檔合成成一個新的映像檔
```
sf read 0x81000000 ${bootA_start} ${bootA_size} # 把目前啟動分區的寫成一個映像檔
run apply_patch 0x81000000 0x80000000 0x82000000 
```
其中的 apply_patch 需要自己實作，但可用 [bsdiff](https://github.com/mendsley/bsdiff) 裡面的 bspatch 直接放在 main 函數中。最後再把生成的新映像檔寫進備份分區且改為啟動分區，然後把目前的啟動分區改為非啟動區，然後重啟即完成整個流程。
```
sf probe 0
sf erase ${bootB_start} ${bootB_size}
sf write 0x82000000 ${bootB_start} ${bootB_size}
setenv boot_partition bootB
saveenv
reset
```
最後的 C 語言版本如下
```C
#include <stdint.h>
#include <stdio.h>
#include "bspatch.h"           // 你移植的 bspatch 標頭檔
#include "spi_flash_driver.h"  // 你自己的 SPI Flash 讀寫驅動

#define OLD_FW_ADDR   0x00000000  // 舊分區起始位址
#define OLD_FW_SIZE   0x00100000  // 舊分區大小（1MB）
#define PATCH_ADDR    0x00200000  // 差分包存放位址
#define PATCH_SIZE    0x00010000  // 差分包大小（64KB）
#define NEW_FW_ADDR   0x00100000  // 新分區起始位址
#define NEW_FW_SIZE   0x00100000  // 新分區大小（1MB）

uint8_t old_fw[OLD_FW_SIZE];
uint8_t patch[PATCH_SIZE];
uint8_t new_fw[NEW_FW_SIZE];

int main(void) {
    int ret;
    spi_flash_read(OLD_FW_ADDR, old_fw, OLD_FW_SIZE); // 1. 讀取舊韌體
    spi_flash_read(PATCH_ADDR, patch, PATCH_SIZE);    // 2. 讀取差分包
    // 3. 使用 bspatch 合成新韌體
    // 假設你已經有一個 bspatch_buffer 版本的 API
    ret = bspatch_buffer(old_fw, OLD_FW_SIZE, patch, PATCH_SIZE, new_fw, NEW_FW_SIZE);
    if (ret != 0) {
        printf("差分還原失敗！\n");
        return -1;
    }
    spi_flash_erase(NEW_FW_ADDR, NEW_FW_SIZE);         // 4. 擦除新分區
    spi_flash_write(NEW_FW_ADDR, new_fw, NEW_FW_SIZE); // 5. 寫進新分區

    // 6. 校驗（可選，建議做 CRC32 或 MD5）
    if (!verify_flash(NEW_FW_ADDR, new_fw, NEW_FW_SIZE)) {
        printf("寫入校驗失敗！\n");
        return -2;
    }
    printf("差分升級成功！\n");
    set_boot_partition(NEW_FW_ADDR);
    mcu_reset();
    return 0;
}
```
```
