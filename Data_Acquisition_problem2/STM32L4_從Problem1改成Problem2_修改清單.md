# 從 Problem 1 改成 Problem 2：修改清單

- 板子：**B-L475E-IOT01A**（STM32L475VGTx），CubeMX 與 CubeIDE 分開使用，macOS
- 前提：你已經做出 **Problem 1 版**（TIM1 觸發 + 單次轉換 + EOC 中斷 + semaphore 通知 task，每筆資料印一行）
- 目標：改成 **Problem 2**——ADC 由 TIM1 定頻觸發，**DMA（循環模式）把資料搬進 buffer**，**buffer 前半填滿**時產生中斷、印出前半，**後半填滿**時產生中斷、印出後半（ping-pong buffer）
- 助教只提供了一份手寫的 `main.c` 骨架（沒有 CubeMX 設定、沒有 `SystemClock_Config`、沒有 UART 與 FreeRTOS 檔案，單獨無法編譯），所以本筆記的做法是：**用 CubeMX + HAL 重建**，再把助教的 TODO 對應補上。

> 標記：**【CubeMX】** 在 CubeMX 做；**【CubeIDE】** 在 CubeIDE 做。

---

## 0. 改之前先備份

把 **Problem 1 的整個專案資料夾（含 `.ioc`）複製成新資料夾**（例如 `Problem2`），**在副本上改**。Problem 1 要 demo，不要動它。

---

## 1. 總覽：改了什麼

| 項目 | Problem 1 | Problem 2 |
|---|---|---|
| 資料傳輸 | 每筆一次 EOC 中斷 | **DMA（Circular）搬進 buffer**，**半滿、全滿**才中斷 |
| ADC 的 DMA Continuous Requests | Disabled | **Enabled** |
| ADC 的 DMA 設定 | 無 | **DMA1 Channel1、Circular、Half Word、記憶體位址遞增** |
| NVIC | ADC1/ADC2 中斷（優先權 5） | 再加 **DMA1 channel1 中斷（優先權 5）** |
| 啟動函式 | `HAL_ADC_Start_IT` | **`HAL_ADC_Start_DMA(&hadc1, buffer, 筆數)`** |
| 資料存放 | 單一變數 `adcValue` | **`sample_buffer[SAMPLE_BUFFER_SIZE]`** |
| 通知機制 | Binary semaphore | **Message queue**（要分辨前半或後半） |
| callback | 只有 `HAL_ADC_ConvCpltCallback` | **`ConvHalfCpltCallback`（前半）＋ `ConvCpltCallback`（後半）** |
| task 一次印 | 1 筆 | **半個 buffer（例如 16 行）** |
| TIM1 | PSC = 8000-1、ARR = 1000-1（10 Hz） | 依 buffer 大小選（見第 4 節） |

**不變的部分**：時脈（HCLK 80 MHz、HSE 關閉）、SYS Timebase = TIM6、ADC 單通道（Temperature Sensor）、640.5 cycles、Scan 與 **Continuous 都 Disabled**、外部觸發 Timer 1 Trigger Out event、USART1（115200 8N1）、FreeRTOS 與 `defaultTask`（Stack 512）、`_write`。

---

## 2. 【CubeMX】修改步驟

開啟**副本專案資料夾裡的 `.ioc`**。

### 步驟 1：ADC1 加回 DMA（順序很重要）

**Analog → ADC1**

1. **DMA Settings** 分頁：按 **Add** → 選 **ADC1**

| 欄位 | 值 |
|---|---|
| Direction | Peripheral To Memory |
| Priority | Low |
| **Mode** | **Circular** |
| Peripheral Increment Address | 不勾 |
| Memory Increment Address | **勾** |
| Data Width（Peripheral、Memory） | **Half Word** |

2. 回 **Parameter Settings** 分頁，把 **DMA Continuous Requests** 設成 **Enabled**（要先做上一步，否則這個選項選不了）
3. 確認這些維持不變：**Scan Disabled、Continuous Conversion Mode Disabled**、Number Of Conversion 1、External Trigger = **Timer 1 Trigger Out event**、Rising edge、通道 Temperature Sensor、640.5 Cycles

> **Circular 與 DMA Continuous Requests 要一起設**：我的理解是，DMA 做循環搬運時，ADC 這一側也要是循環模式（DMA Continuous Requests = Enabled），否則 DMA 搬完設定的筆數後 ADC 就不再發 DMA 請求，現象會是**只印出前半、後半各一次就不再有輸出**。這是我讀手冊後的理解，不確定的話以你實測 top／bottom 是否持續交替為準。
>
> **Continuous Conversion Mode 要 Disabled**：連續模式下外部觸發只啟動第一次，之後 ADC 自己連續轉換，取樣頻率就不是 Timer 設的頻率了。

### 步驟 2：NVIC

**System Core → NVIC**：

- **DMA1 channel1 global interrupt**：勾選、**Preemption Priority 設 5**
- **ADC1 and ADC2 interrupts**：維持勾選、優先權 5

> 為什麼是 5：callback 是在 DMA 中斷裡被呼叫的，裡面會呼叫 RTOS 函式（`osMessageQueuePut`），FreeRTOS 規定這類中斷的優先權數值不能小於 `configMAX_SYSCALL_INTERRUPT_PRIORITY`（預設 5，數字越小優先權越高）。助教的骨架把 DMA 中斷優先權設成 0，這樣會有問題，要改成 5。

### 步驟 3：把 semaphore 換成 queue

**Middleware and Software Packs → FREERTOS → Timers and Semaphores / Queues**

1. （選用）刪掉 Problem 1 的 `adcsem` binary semaphore（不刪也不影響，但程式裡就不會用到它了）
2. **Tasks and Queues** 分頁 → **Queues** 區塊 → **Add**：

| 欄位 | 值 |
|---|---|
| Queue Name | 例如 `adcqueue`（**程式裡的 handle = 名稱 + `Handle`**，填 `adcqueue` 就是 `adcqueueHandle`，大小寫要完全一致） |
| Queue Size | 4 |
| Item Size | `uint8_t`（只需要放一個 0 或 1） |
| Allocation | Dynamic |

> **為什麼用 queue 而不是 semaphore**：task 要分辨「前半好了」還是「後半好了」，queue 可以帶一個值（0 = 前半，1 = 後半）；而且 binary semaphore 不累積計數，兩個通知緊接著發生時可能被合併，queue 不會。

### 步驟 4：TIM1 頻率（看第 4 節的選擇）

**Timers → TIM1 → Parameter Settings**：

| 想要 | PSC | ARR | 頻率 |
|---|---|---|---|
| 預設（buffer 256，每次印 128 行） | 8000-1 | 100-1 | 100 Hz |
| **每次只印約十幾行（建議）** | 8000-1 | **1000-1** | **10 Hz** |

### 步驟 5：產生程式碼

**GENERATE CODE**（輸出到副本專案資料夾）。

---

## 3. 【CubeIDE】修改程式碼

先在專案按右鍵 **Refresh（F5）**。以下都寫在 `main.c` 的 `USER CODE` 區塊內。

### ① `USER CODE BEGIN PV`

```c
// 刪除（Problem 1）：volatile uint16_t adcValue;

// 改成：
#define SAMPLE_BUFFER_SIZE  32       // 想每次印約十幾行：32（每半 16 筆）；預設 256（每半 128 筆）
uint16_t sample_buffer[SAMPLE_BUFFER_SIZE];
```

- buffer 型別用 `uint16_t`，對應 DMA 設定的 **Half Word**。
- `SAMPLE_BUFFER_SIZE` 要**保持偶數**，前半與後半才一樣大；每次印的行數 = `SAMPLE_BUFFER_SIZE / 2`。
- 因為題目是印出「半個 buffer 的所有資料」，想要每次行數少一點，就要**縮小 buffer**，而不是只印一部分。

### ② `USER CODE BEGIN Includes`（確認有）

```c
#include <stdio.h>
```

### ③ `main()` 的 `USER CODE BEGIN 2`：保持清空

```c
/* USER CODE BEGIN 2 */
/* USER CODE END 2 */
```

### ④ `USER CODE BEGIN 4`：保留 `_write`，兩個 callback

```c
int _write(int file, char *ptr, int len)       // 保留，讓 printf 輸出到 UART1
{
    HAL_UART_Transmit(&huart1, (uint8_t*)ptr, len, HAL_MAX_DELAY);
    return len;
}

void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef* hadc)    // 前半 (top half) 填滿
{
    if (hadc->Instance == ADC1)
    {
        uint8_t msg = 0;
        osMessageQueuePut(adcqueueHandle, &msg, 0, 0);        // 在 ISR 裡 timeout 必須是 0
    }
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)        // 後半 (bottom half) 填滿
{
    if (hadc->Instance == ADC1)
    {
        uint8_t msg = 1;
        osMessageQueuePut(adcqueueHandle, &msg, 0, 0);
    }
}
```

> **為什麼 `ConvCpltCallback` 是「後半」**：DMA 傳輸到 buffer 一半時產生 Half Transfer 中斷（呼叫 `ConvHalfCpltCallback`），傳輸到結尾時產生 Transfer Complete 中斷（呼叫 `ConvCpltCallback`）。Circular 模式下 DMA 結尾後自動回到開頭，兩個中斷輪流發生。

### ⑤ `StartDefaultTask` 的 `USER CODE BEGIN 5`：整段換掉

```c
HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED);                      // 1. 校準（必須在啟動 ADC 之前）
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)sample_buffer, SAMPLE_BUFFER_SIZE);    // 2. 啟動 ADC + DMA（長度是「筆數」，不是位元組數）
HAL_TIM_Base_Start(&htim1);                                                 // 3. 啟動 TIM1，開始定頻觸發

for(;;)
{
    uint8_t msg;
    osMessageQueueGet(adcqueueHandle, &msg, NULL, osWaitForever);           // 睡到有人通知

    uint16_t *p = &sample_buffer[msg ? SAMPLE_BUFFER_SIZE / 2 : 0];         // 0 = 前半，1 = 後半
    printf("--- %s half ---\r\n", msg ? "bottom" : "top");
    for (int i = 0; i < SAMPLE_BUFFER_SIZE / 2; i++)
    {
        int32_t temp = __HAL_ADC_CALC_TEMPERATURE(3300, p[i], ADC_RESOLUTION_12B);
        printf("%u, %ld C\r\n", p[i], (long)temp);
    }
}
```

### 資料流（Problem 2）

```
TIM1 定頻 Update Event → TRGO → ADC1 轉換溫度感測器（單次）
   ↓ ADC 發 DMA request
DMA1 Channel1 把結果搬進 sample_buffer（Circular，不經過 CPU）
   │
   ├ 前半填滿 → Half Transfer 中斷 → ConvHalfCpltCallback → queue 放 0
   │       → task 印前半
   └ 後半填滿 → Transfer Complete 中斷 → ConvCpltCallback → queue 放 1
           → task 印後半
   ↓ DMA 回到 buffer 開頭，循環
```

DMA 在填後半時，task 在印前半，互不覆蓋（ping-pong buffer）。

---

## 4. buffer 大小與取樣頻率怎麼選

填滿半個 buffer 的時間 = (`SAMPLE_BUFFER_SIZE` / 2) ÷ 取樣頻率；task 印半個 buffer 的時間約 (`SAMPLE_BUFFER_SIZE` / 2) × 1 ms（115200 baud，每行約 11 個字元）。

**限制：印完的時間必須小於填滿另一半的時間**，否則 task 還在印前半時，DMA 已經繞回來把前半覆蓋掉。

| buffer | 取樣頻率 | 每半筆數 | 填滿半個 | 印半個 | 結論 |
|---|---|---|---|---|---|
| 256 | 100 Hz | 128 | 1.28 s | 約 0.14 s | 安全，但一次刷 128 行 |
| **32** | **10 Hz** | **16** | **1.6 s** | **約 17 ms** | **安全，輸出好讀（建議）** |
| 32 | 100 Hz | 16 | 0.16 s | 約 17 ms | 安全，但畫面捲得太快 |
| 256 | 1 kHz | 128 | 128 ms | 約 0.14 s | **印不完，資料會被覆蓋** |

另外 queue 大小是 4：如果 task 長期來不及處理，queue 滿了之後 `osMessageQueuePut` 會失敗（回傳錯誤、通知遺失），也是資料遺失的徵兆。

---

## 5. 編譯、燒錄、驗證

1. **Build**，確認 0 errors
2. **Run** 燒錄；序列終端機連線 115200 8N1
3. 預期（以 buffer 32、10 Hz 為例）每 1.6 秒出現一塊：

```
--- top half ---
946, 29 C
945, 29 C
...（共 16 行）
--- bottom half ---
946, 29 C
...（共 16 行）
--- top half ---
...
```

| 檢查 | 預期 | 如果不是 |
|---|---|---|
| top、bottom 交替 | 持續交替，不會連續兩個相同 | **只出現一輪就停**：DMA Continuous Requests 沒有 Enabled，或 DMA 沒有設 Circular |
| 每塊行數 | 正好 `SAMPLE_BUFFER_SIZE / 2` 行 | |
| 標題間隔 | 約 1.6 s（10 Hz、buffer 32） | 太快：Continuous 沒有 Disabled，或 TIM1 設定不對 |
| 溫度 | 約 20～45 °C，raw 在幾個值之間小幅跳動 | |
| 完全沒輸出 | | 檢查 `_write`、`#include <stdio.h>`、序列埠、USB 接 CN7；用 Debug 暫停看是否卡在 `Error_Handler` |

> 提醒：**DMA Continuous Requests + Circular 這組設定是否正確，要靠實測 top／bottom 是否持續交替來確認**；我對它的理解來自手冊與 HAL 的說明，沒有替你的板子保證。

---

## 6. 助教骨架 TODO 對照表

| 助教的 TODO | 本做法 |
|---|---|
| declare RTOS object to commute between ISR and RTOS task | CubeMX 的 queue `adcqueue`（產生 `adcqueueHandle`） |
| `HAL_ADC_ConvCpltCallback`：inform RTOS task | 放 1（後半） |
| `HAL_ADC_ConvHalfCpltCallback`：inform RTOS task | 放 0（前半） |
| define the RTOS task function | `StartDefaultTask` 的迴圈 |
| init UART for printing | CubeMX 的 USART1（PB6／PB7，115200 8N1），加 `_write` |
| init some RTOS object and task | CubeMX 自動產生 |
| `HAL_ADC_Start_DMA(...)` 的參數 | `(&hadc1, (uint32_t*)sample_buffer, SAMPLE_BUFFER_SIZE)` |
| start RTOS scheduling | CubeMX 產生的 `osKernelStart()` |

### 助教骨架裡我看到、建議留意的地方

| 位置 | 內容 | 說明 |
|---|---|---|
| `HAL_NVIC_SetPriority(DMA1_Channel1_IRQn, 0, 0)` | DMA 中斷優先權 0 | 會呼叫 RTOS 函式的中斷優先權數值要 ≥ 5，我用 5 |
| `ADC_SAMPLETIME_2CYCLES_5` | 取樣時間 2.5 cycles | 溫度感測器輸出阻抗高，需要較長取樣時間；我沿用 Problem 1 的 640.5 cycles。想知道助教為何選 2.5，可以直接問 |
| `ADC_CLOCK_SYNC_PCLK_DIV4` | ADC 用同步時脈 | 我沿用 Problem 1 的非同步時脈（CubeMX 預設） |
| `DMA1_Channel1_IRQHandler` 內 `SET_BIT(hadc1.Instance->CFGR, ADC_CFGR_DMACFG)` | 手動寫 ADC 暫存器 | 我的理解是這個位元決定 ADC 的 DMA 是單次還是循環；HAL 的 `DMAContinuousRequests = ENABLE` 初始化時應該會設同一個位元。這是我的推測，沒有查證 |
| `NVIC_SetVector(...)` | 改 RAM 向量表 | CubeIDE 專案預設向量表在 Flash，我沒有使用這行 |

### 要問助教的兩件事

題目說「HAL 無法完全實現，需透過 LL」，但骨架幾乎全是 HAL，我看不出 HAL 缺什麼。

1. 用 **CubeMX + HAL** 做出來的版本可以接受嗎？還是一定要保留他給的手寫結構（或要用 LL 取代某一段）？
2. **HAL 具體做不到哪一件事？**

---

## 7. 向助教說明參數（重點版）

**一句話版**：TIM1 定頻 Update Event 經 TRGO 觸發 ADC1 單次轉換，ADC 發 DMA 請求，DMA（循環模式）把結果搬進 `sample_buffer`；buffer 填到一半時 DMA 產生 Half Transfer 中斷、填滿時產生 Transfer Complete 中斷，ISR 只往 queue 放一個值（0 = 前半、1 = 後半）通知 task，task 再把對應的那半個 buffer 用 UART 印出。

| 參數 | 設定 | 理由 |
|---|---|---|
| 為什麼用 DMA | | 資料由硬體搬進 buffer，CPU 不用每筆都被打斷；半滿、全滿才中斷，中斷次數降為 Problem 1 的 1/（每半筆數） |
| DMA 模式 | Circular、Half Word、記憶體位址遞增、週邊位址不遞增 | 循環讓 buffer 用完自動回到開頭；ADC 結果 12 位元，Half Word（16 位元）剛好；ADC 資料暫存器位址固定，所以週邊位址不遞增 |
| ADC DMA Continuous Requests | Enabled | 讓 ADC 在 DMA 循環時持續發出請求 |
| Continuous Conversion | Disabled | 連續模式下外部觸發只啟動第一次，無法固定頻率 |
| buffer 大小 | 32（每半 16 筆） | 每次印約 16 行方便觀察；題目要印「半個 buffer 的所有資料」，所以用縮小 buffer 控制行數 |
| 取樣頻率 | 10 Hz（PSC = 8000-1、ARR = 1000-1，80 MHz） | 填滿半個 buffer 1.6 s，印 16 行約 17 ms，遠小於 1.6 s，ping-pong 不會互相覆蓋；輸出不會捲太快 |
| 為什麼用 queue | | 要分辨前半或後半；queue 可累積通知，不像 binary semaphore 會合併 |
| ISR 只放 queue | | ISR 會搶佔所有 task，UART 印字慢，所以印字放在 task |
| NVIC 優先權 5 | DMA 與 ADC 中斷 | 會呼叫 RTOS API 的中斷，數值不能小於 `configMAX_SYSCALL_INTERRUPT_PRIORITY` |
| ping-pong 的時間限制 | | 印完半個 buffer 的時間必須小於 DMA 填滿另一半的時間，否則資料被覆蓋 |

**可能被追問的**：

- **頻率提高到 1 kHz 會怎樣？** buffer 256 時填滿半個只要 128 ms，印 128 行約 0.14 s，印不完，資料被覆蓋、queue 也可能滿。解法是降低輸出量（例如只印統計值）、加大 buffer，或提高 UART 速度。
- **Problem 1 與 Problem 2 的取捨？** Problem 1 簡單，每筆一次中斷，適合低速；Problem 2 用 DMA 減少中斷次數，適合高速，但要處理 buffer 同步與時間限制。
- **為什麼 callback 裡 `osMessageQueuePut` 的 timeout 是 0？** ISR 裡不能阻塞，timeout 必須為 0。

**請自己查證再回答的項目**（我憑印象，沒有驗證）：
- ADC 的 DMA 單次／循環設定（CFGR 的 DMACFG 位元）在 HAL 裡是否由 `DMAContinuousRequests` 設定
- 內部溫度感測器在 datasheet 的最小取樣時間（我印象中約 5 µs）
- `__HAL_ADC_CALC_TEMPERATURE` 的實作原理（我印象中用出廠校準值線性內插，假設供電 3300 mV）

---

## 8. 這次改的過程中踩過的坑

| 現象 | 原因 / 處理 |
|---|---|
| 編譯錯誤：`adcQueueHandle` 未宣告 | CubeMX 產生的是你填的 queue 名稱 + `Handle`（填 `adcqueue` 就是 `adcqueueHandle`），程式裡大小寫要完全一致 |
| 終端機完全沒輸出（編譯卻沒錯） | 副本專案缺 `_write` 或 `#include <stdio.h>`；確認 `USER CODE BEGIN 4` 裡有 `_write` |
| 輸出狂刷、不是預期的間隔 | **Continuous Conversion Mode 是 Enabled**，改成 Disabled |
| 只印出 top、bottom 各一次就停 | **DMA Continuous Requests 是 Disabled**，改成 Enabled（要先設 DMA Circular） |
| `DMA Continuous Requests` 選不了 Enabled | 要先在 DMA Settings 分頁 Add ADC1 並設成 Circular |
| 開機卡死 | 用 Debug 模式暫停，看是不是停在 `Error_Handler`；檢查 DMA 與 ADC 中斷優先權是 5，不是 0 |
| 重新產生程式碼後 CubeIDE 沒更新 | 專案按右鍵 Refresh（F5） |
| `MX_GPIO_Init` 還有 `ARD_D4_Pin` 設成 `GPIO_AF1_TIM2` | TIM2 留下的腳位設定，不影響功能（想清乾淨就在 Pinout 點 PA3 選 Reset_State） |

---

## 9. 小提醒

- 回 CubeMX 改設定、重新產生程式碼之前，確認你的程式碼都寫在 `USER CODE BEGIN … END` 之間。
- `.ioc` 一定要開**專案資料夾裡的那一份**。
- 改頻率只要改 TIM1 的 PSC、ARR（兩者都不能超過 65535，TIM1 是 16 位元）；改 buffer 大小只要改 `SAMPLE_BUFFER_SIZE`（保持偶數）。兩者改完都要重新檢查第 4 節的時間限制。
