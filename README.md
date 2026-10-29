# HW2：Zynq-7020 中斷控制器研究與 LED / 按鈕中斷驅動實作

> 以 **EGO-XZ7（Zynq-7020）** 為平台，從查閱官方手冊、整理 GIC 中斷暫存器，到完成 **C → AXI GPIO → LED** 以及 **按鈕 → GIC → ISR → LED** 的完整軟硬體整合。

## 目錄

1. [專案概述](#1-專案概述)
2. [一：手冊中 BSP / ISR 相關章節](#2-一手冊中-bsp--isr-相關章節)
3. [二：UG585 中與中斷相關的暫存器](#3-二ug585-中與中斷相關的暫存器)
4. [三：實作成果](#4-三實作成果)
5. [實驗結果與驗證](#5-實驗結果與驗證)
6. [問題與心得](#6-問題與心得)
7. [如何重現](#7-如何重現)
8. [參考資料](#8-參考資料)

---

## 1. 專案概述

| 項目 | 內容 |
| --- | --- |
| **開發板** | E-Elements EGO-XZ7（Zynq-7020，XC7Z020-1CLG484，雙核 Cortex-A9） |
| **工具版本** | Vivado 2018.2 / 2018.3、Xilinx SDK 2018.2 |
| **程式語言** | C（Bare-metal / standalone BSP） |
| **作業目標** | ① 查找 Zynq-7020 手冊中 BSP / ISR 相關章節 <br> ② 查找 UG585 中與中斷相關且具實體位址的暫存器（IRQ、Pending、Mask） <br> ③ 實作：(a) 點亮 LED（C → FPGA → LED）(b) 按按鈕觸發中斷改變 LED 狀態（Driver + ISR） |

---

## 2. 作業一：手冊中 BSP / ISR 相關章節

| 類型 | 文件 |
| --- | --- |
| **User Guide** | **UG585** — *Zynq-7000 SoC Technical Reference Manual* (v1.14) |
| **Programming Guide** | **UG821** — *Zynq-7000 SoC Software Developers Guide* (v12.0) |

### 2.1 BSP 相關章節（UG821）

| 章節 | 頁碼 | 重點 |
| --- | --- | --- |
| Ch.2 Software Application Development Flows → Software Tools Overview | p.10–12 | Vivado 將 PS 設定與 PL 資訊匯出為硬體描述檔（`.hdf`），SDK 依此建立並設定 BSP。 |
| Ch.2 → Bare-Metal Device Driver Architecture | p.14–17 | standalone 驅動分層：Layer 2（RTOS Adapter）、Layer 1（Device Driver API）與 Direct Hardware Interface（暫存器位址常數與讀寫巨集）。 |
| Ch.2 → Bare-Metal Application Development | p.17–19 | 匯入硬體平台、建立 BSP、建立應用程式、編譯與燒錄的完整流程。 |

**本作業用到的 BSP 核心檔案：**

| 檔案 | 用途 |
| --- | --- |
| `xparameters.h` | 定義各 IP 的實體位址與中斷編號，例如 `XPAR_AXI_GPIO_0_BASEADDR = 0x41200000`、`XPAR_FABRIC_AXI_GPIO_1_IP2INTC_IRPT_INTR = 61`。 |
| `xil_io.h` | 提供 `Xil_In32` / `Xil_Out32`，直接讀寫 GIC 與 GPIO 暫存器。 |
| `xil_exception.h` | 提供 `Xil_ExceptionRegisterHandler`、`Xil_ExceptionEnable`，將 ISR 掛載至 IRQ 例外向量。 |

### 2.2 ISR 相關章節（UG585）

| 章節 | 頁碼 | 重點 |
| --- | --- | --- |
| Ch.7 Interrupts → Environment & GIC | p.231–234 | SGI、PPI、SPI 三種中斷類型與 GIC（Generic Interrupt Controller）方塊圖。 |
| Ch.7 → Functional Description | p.235–238 | 各中斷來源對應的 IRQ ID（Table 7-2 ~ 7-4）。 |
| Ch.7 → Programming Model → Interrupt Prioritization & Handling | p.243–244 | GIC 如何挑選最高優先權中斷送給 CPU，以及 ISR 的正確處理順序。 |

> **規範重點：** ISR 必須先讀取 GIC 暫存器確認中斷來源，再處理周邊的狀態暫存器，最後通知 GIC 中斷結束。本作業的 `btn_isr()` 依此順序實作：先讀 `ICCIAR` 確認 ID 61 → 處理 AXI GPIO 並清除其中斷 → 寫 `ICCEOIR`。

---

## 3. 二：UG585 中與中斷相關的暫存器

### 3.1 中斷傳遞路徑

```
按鈕 ─► axi_gpio_1 (ip2intc_irpt) ─► IRQ_F2P[0] ─► GIC Distributor ─► GIC CPU Interface ─► CPU (IRQ)
```

GIC 分為兩大區塊：

| 區塊 | 縮寫 | 基底位址 | 職責 |
| --- | --- | --- | --- |
| **Distributor** | ICD | `0xF8F0_1000` | 決定每個中斷是否啟用、優先權、目標 CPU、觸發方式（level / edge）。 |
| **CPU Interface** | ICC | `0xF8F0_0100` | 將中斷交付給 CPU；CPU 由此讀取中斷編號，處理完畢後回報。 |

### 3.2 作業要求的三項暫存器

| 項目 | 暫存器 | 實體位址 | 說明 |
| --- | --- | --- | --- |
| **1. IRQ #（Port）** | `IRQ_F2P[0]` | — | PL → PS 中斷埠，對應 **IRQ ID = 61**（UG585 Table 7-4）。 |
| **2. Pending（Status）** | `ICDISPR1` / `ICDICPR1` | `0xF8F0_1204` / `0xF8F0_1284` | **bit 29**（ID 61 = 32 + 29）。讀取可查看 pending 狀態；對 `ICDICPR1` 寫 1 可清除殘留 pending。 |
| **3. Mask（Enable）** | `ICDISER1` / `ICDICER1` | `0xF8F0_1104` / `0xF8F0_1184` | **bit 29**。對 `ICDISER1` 寫 1 啟用 ID 61；對 `ICDICER1` 寫 1 則停用。 |

> 位址推導：ID 32–63 位於各暫存器組的第 2 個字（index 1），故為基底 + `0x4`。ID 61 在該字中的 bit 位置為 61 − 32 = **29**。

### 3.3 其他用到的 GIC / 周邊暫存器

| 暫存器 | 實體位址 | 設定值 / 功能 |
| --- | --- | --- |
| `ICDDCR`（Distributor Control） | `0xF8F0_1000` | 寫 `1` 啟用 Distributor。 |
| `ICDICFR3`（觸發方式） | `0xF8F0_1C0C` | `bit[27:26] = 01`，將 ID 61 設為 High-Level 觸發。 |
| `ICCICR`（CPU Interface Control） | `0xF8F0_0100` | `0x07`，啟用 CPU Interface。 |
| `ICCPMR`（優先權門檻） | `0xF8F0_0104` | `0xF0`，優先權數值小於門檻的中斷才會送達 CPU。 |
| `ICCIAR`（中斷應答） | `0xF8F0_010C` | ISR 開頭讀取，取得中斷 ID，同時代表 CPU 已接收。 |
| `ICCEOIR`（中斷結束） | `0xF8F0_0110` | ISR 結尾寫入該 ID，通知 GIC 處理完畢。 |
| AXI GPIO `IPISR` | `0x4121_0120` | AXI GPIO 中斷狀態，**寫 1 清除**（toggle-on-write）。 |

---

## 4. 三：實作成果

### 4.1 硬體架構（Vivado Block Design）

| 元件 | 設定 | 用途 |
| --- | --- | --- |
| `processing_system7_0` | 啟用 `M_AXI_GP0`；啟用 `IRQ_F2P[0:0]` | CPU 存取 PL 周邊；PL 中斷入口 |
| `axi_gpio_0` | 8-bit、**Outputs** | 連接 8 顆板載 LED |
| `axi_gpio_1` | 1-bit、**Inputs**、啟用 **Enable Interrupt** | 接收按鈕；`ip2intc_irpt` 接至 `IRQ_F2P[0]` |

**位址分配（Address Editor）：**

| 元件 | 基底位址 | 大小 |
| --- | --- | --- |
| `axi_gpio_0`（LED） | `0x4120_0000` | 64K |
| `axi_gpio_1`（按鈕） | `0x4121_0000` | 64K |

> 📷 *Block Design 截圖：* `docs/block_design.png`

### 4.2 軟體設計（`sdk/main.c`）

**LED 行為：** 8 顆 LED 預設執行跑馬燈（每 0.2 秒移動一格）；按下按鈕觸發中斷，ISR 反轉 `paused` 旗標，使跑馬燈暫停／繼續。

**初始化流程：**

1. 設定 GPIO 方向（LED 為輸出、按鈕為輸入）。
2. 初始化 GIC：設定 ID 61 的優先權、目標 CPU、High-Level 觸發；清除殘留 pending；啟用 Mask。
3. 啟用 AXI GPIO 中斷（`GIER` 全域致能、`IPIER` 通道致能）。
4. 以 `Xil_ExceptionRegisterHandler` 將 `btn_isr` 註冊至 IRQ 例外向量，並呼叫 `Xil_ExceptionEnable()` 致能 CPU 例外。

**ISR（`btn_isr`）處理順序：**

| 步驟 | 動作 | 原因 |
| --- | --- | --- |
| 1 | 讀取 `ICCIAR`，確認中斷 ID 為 61 | 確認來源並向 GIC 應答 |
| 2 | 讀取 GPIO 資料暫存器，反轉 `paused` 旗標 | 執行按鈕對應的應用邏輯 |
| 3 | 對 AXI GPIO `IPISR` 寫 1 清除 | Level 觸發若不清除，中斷線持續為高，會陷入無窮中斷迴圈 |
| 4 | 寫回 `ICCEOIR` | 通知 GIC 此中斷已處理完畢 |

**核心流程示意**（請以實際 `main.c` 為準）：

```c
/* ---- GIC 設定 (ID 61 = bit 29 of register index 1) ---- */
Xil_Out32(0xF8F01000, 0x1);                                   /* ICDDCR: enable distributor */
Xil_Out32(0xF8F01284, 1u << 29);                              /* ICDICPR1: clear stale pending */
Xil_Out32(0xF8F01104, 1u << 29);                              /* ICDISER1: enable ID 61 */
Xil_Out32(0xF8F00104, 0xF0);                                  /* ICCPMR: priority mask */
Xil_Out32(0xF8F00100, 0x07);                                  /* ICCICR: enable CPU interface */

/* ---- ISR ---- */
void btn_isr(void *ref)
{
    u32 id = Xil_In32(0xF8F0010C) & 0x3FF;     /* ICCIAR: acknowledge */
    if (id == 61) {
        paused = !paused;                       /* application logic */
        Xil_Out32(0x41210120, 0x1);             /* IPISR: write-1-to-clear */
    }
    Xil_Out32(0xF8F00110, id);                  /* ICCEOIR: end of interrupt */
}
```

---

## 5. 實驗結果與驗證

| 測試項目 | 預期結果 | 結果 |
| --- | --- | --- |
| 上電後 LED | 8 顆 LED 跑馬燈，每 0.2 秒移動一格 | ✅ |
| 按下按鈕 | 觸發 ID 61 中斷，跑馬燈暫停 | ✅ |
| 再按一次 | 跑馬燈繼續 | ✅ |
| 長按按鈕 | 不應卡死（`IPISR` 已正確清除） | ✅ |

> 📷 *實機照片／Demo 影片連結：* `docs/demo.jpg`、`<影片網址>`

---

## 6. 問題與心得

* **Level 觸發必須清除周邊狀態：** 若 ISR 只寫 `ICCEOIR` 而未清除 AXI GPIO 的 `IPISR`，中斷線仍為高，離開 ISR 後立即再次觸發，系統卡死。
* **ISR 順序有規範：** 先 `ICCIAR`（應答）→ 處理周邊 → `ICCEOIR`（結束），順序不可顛倒。
* **ID 與 bit 位置換算：** SPI 的 ID 32 起算，ID 61 位於第 2 組暫存器的 bit 29（61 − 32）；觸發方式 `ICDICFR` 每個暫存器涵蓋 16 個 ID、每個 ID 占 2 bits，ID 61 落在 `ICDICFR3` 的 bit[27:26]。
* **BSP 是橋樑：** `.hdf` 匯出後由 SDK 自動產生 `xparameters.h`，硬體位址改變時軟體不需手動修改常數。

---

## 7. 如何重現

1. 以 Vivado 2018.2 開啟專案，確認 Block Design 與 Address Editor 如 §4.1。
2. 產生 Bitstream，**File → Export → Export Hardware（勾選 Include bitstream）**，再 **Launch SDK**。
3. 在 SDK 建立 Application Project（standalone BSP），將 `sdk/main.c` 加入並編譯。
4. 連接開發板，**Xilinx → Program FPGA**，再 **Run As → Launch on Hardware (System Debugger)**。

### 專案結構

```
.
├── README.md
├── vivado/          # Block Design / 約束檔 / 專案腳本
├── sdk/
│   └── main.c       # LED 跑馬燈 + 按鈕 ISR
└── docs/            # 截圖與 Demo
```

---

## 8. 參考資料

* Xilinx **UG585** — *Zynq-7000 SoC Technical Reference Manual* (v1.14), Ch.7 Interrupts
* Xilinx **UG821** — *Zynq-7000 SoC Software Developers Guide* (v12.0), Ch.2
* Xilinx **PG144** — *AXI GPIO LogiCORE IP Product Guide*（`GIER` / `IPIER` / `IPISR` 暫存器）
* ARM *Generic Interrupt Controller Architecture Specification*
