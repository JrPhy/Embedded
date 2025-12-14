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
                    mmc dev 0; \
# 選擇 eMMC 裝置編號 0，準備操作。
                    mmc erase 0x1000 0x8000; \
# 擦除 eMMC 上從位址 0x1000 開始、長度 0x8000 的區塊。這裡代表要清掉舊韌體所在的分區。
                    mmc write 0x80000000 0x1000 0x8000; \
# 把剛剛下載到 RAM 的映像檔，寫入到 eMMC 的分區（起始位址 0x1000，大小 0x8000）。
                    reset"
# 完成更新後，重啟系統，讓新韌體生效。
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
會去比較新舊 BIN 檔然後生成 uboot.diff 這個更新包，接著在 uboot 輸入已下指令就可以更新了
```
sf probe 0
sf erase <B分區offset> <size>
sf write <RAM地址> <B分區offset> <size>
```
當更新包下載到 device 後，會利用啟動分區的檔案與更新包，生成一個新的安裝包，然後再依照整包更新的流程去做，所以其他流程都與前面無異。
