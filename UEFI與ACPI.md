不同於 linux，windows 長期以來是在 x86 架構下發展，近年來雖然也有開發 Arm 架構下的 OS，但還是依照 x86 的開發方式。而電腦中的 BIOS 就如同 linux 的 bootloader，做的事情一樣但多了幾個步驟，不過近年來因為平板電腦的興起，倆著的界線也漸漸變得模糊

### BIOS vs. UEFI vs. U-Boot 綜合對比表

| 特性 | **傳統 BIOS (Legacy)** | **UEFI** | **U-Boot** |
| :--- | :--- | :--- | :--- |
| **主要定位** | 早期 PC 啟動韌體 | 現代化通用韌體標準 (PC/Server) | 嵌入式系統引導程式 (IoT/Mobile) |
| **開發組織** | 各大 BIOS 廠商 (如 Phoenix, AMI) | UEFI Forum (Intel, MS, AMD 等) | 開源社群 (Denx Software Engineering) |
| **處理器架構** | 僅限 x86 (16-bit 模式) | x86, ARM, RISC-V (32/64-bit) | 廣泛支援 (ARM, RISC-V, MIPS, PPC) |
| **硬體描述方式** | 硬編碼或透過 Interrupts | **ACPI** Table | **Device Tree (DTB)** |
| **磁碟分割支援** | MBR (最大 2.2 TB) | **GPT** (最大支援 9.4 ZB) | 支援多種 (MBR, GPT, Raw Flash) |
| **啟動協議** | 讀取磁區 (Int 19h) | 載入 **.efi 檔案** (FAT32 分割區) | 載入 Kernel 映像檔 (uImage/fitImage) |
| **安全性** | 無 (易受 MBR 病毒攻擊) | **Secure Boot** (數位簽章驗證) | Verified Boot (需配合硬體 TrustZone) |
| **使用者介面** | 藍底白字 / 僅限鍵盤 | 圖形化 GUI / 支援滑鼠操作 | 強大的命令行 (CLI) 交互 |
| **開機速度** | 慢 (需逐一列舉硬體) | 快 (支援硬體並行初始化) | 極快 (高度優化與裁剪) |
| **擴充性** | 極低 | 高 (可載入驅動與應用程式) | 中 (透過腳本與模組化配置) |
