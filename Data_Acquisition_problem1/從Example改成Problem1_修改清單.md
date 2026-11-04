# 從講義 Example 改成 Problem 1：修改清單

- 板子：**B-L475E-IOT01A**（STM32L475VGTx），CubeMX 與 CubeIDE 分開使用，macOS
- 前提：你已經照《STM32L4_ADC_Timer_DMA_FreeRTOS_實作筆記》做出 **Example 版**（TIM2 觸發 + DMA + 旗標輪詢，每秒一行）
- 目標：改成 **Problem 1**——ADC1 由 **TIM1** 定頻觸發，每次轉換完成產生**中斷**，由中斷通知 **RTOS task**，task 透過 UART 印出
- 結果：已在板子上跑通，序列終端機持續輸出 `raw = ..., temp = .. C`

> 標記：**【CubeMX】** 在 CubeMX 做；**【CubeIDE】** 在 CubeIDE 做。

---

## 0. 改之前先備份

把**整個專案資料夾**複製一份（包含 `.ioc`），改壞了還能回到「每秒一行」的可運作版本。

---

## 1. 總覽：改了什麼

| 項目 | Example 版 | Problem 1 |
|---|---|---|
| 觸發 Timer | TIM2 | **TIM1** |
| 觸發頻率 | 1 Hz（PSC = 80-1、ARR = 1000000-1） | **10 Hz**（PSC = 8000-1、ARR = 1000-1，自己決定） |
| ADC 觸發來源 | Timer 2 Trigger Out event | **Timer 1 Trigger Out event** |
| 資料傳輸 | DMA（Circular） | **中斷**（EOC 中斷，不用 DMA） |
| ADC 的 DMA Continuous Requests | Enabled | **Disabled** |
| 啟動函式 | `HAL_ADC_Start_DMA` | **`HAL_ADC_Start_IT`** |
| 通知 task 的方式 | 全域旗標 `adcDone` + task 每 100 ms 輪詢 | **Binary semaphore** |
| 啟動動作放哪 | `main()` 的 `USER CODE BEGIN 2` | **task 裡**（`StartDefaultTask`） |
| 結果變數 | `adcRaw[1]`（DMA 搬進來） | `adcValue`（callback 用 `HAL_ADC_GetValue` 讀） |

**不變的部分**：時脈（HCLK 80 MHz、HSE 關閉）、SYS Timebase = TIM6、ADC 單通道（Temperature Sensor）、640.5 cycles、Scan 與 Continuous 都 Disabled、USART1（115200 8N1）、FreeRTOS 與 `defaultTask`（Stack 512）。

---

## 2. 【CubeMX】修改步驟

開啟**專案資料夾裡的 `.ioc`**（不是另外存的副本）。

### 步驟 1：設定 TIM1

**Timers → TIM1**，Clock Source 選 **Internal Clock**，Parameter Settings：

| 欄位 | 值 |
|---|---|
| Prescaler | `8000-1` |
| Counter Period | `1000-1` |
| auto-reload preload | Disable |
| Trigger Event Selection **TRGO** | **Update Event** |
| Trigger Event Selection TRGO2 | 保持預設 **Reset**（不用設，ADC 選的是 TRGO） |

TIM1 的 Channel1～4 全部保持 Disable（不需要腳位）。

頻率計算：

```
f = 80 MHz / ((PSC+1) × (ARR+1)) = 80 MHz / (8000 × 1000) = 10 Hz
```

TIM1 的 PSC、ARR 都是 **16 位元**（CubeMX 欄位會標示 "16 bits value"），最大 65535，所以降頻要拆成兩個欄位相乘，不能像 TIM2（32 位元）那樣把 ARR 填 1000000-1。

### 步驟 2：關掉 TIM2

**Timers → TIM2**，Clock Source 改回 **Disable**。（避免殘留 `htim2`、`MX_TIM2_Init` 造成混淆。）

### 步驟 3：修改 ADC1（順序很重要）

**Analog → ADC1**

1. **Parameter Settings**：先把 **DMA Continuous Requests** 改成 **Disabled**
2. **DMA Settings** 分頁：選取 ADC1 那一列，按 **Delete**
3. 回 **Parameter Settings**：**External Trigger Conversion Source** 改成 **Timer 1 Trigger Out event**（**不要選** "Timer 1 Trigger Out 2 event"，那是 TRGO2），Edge 維持 **Rising edge**
4. 確認這些維持不變：

| 欄位 | 值 |
|---|---|
| Scan Conversion Mode | Disabled |
| **Continuous Conversion Mode** | **Disabled** |
| Number Of Conversion | 1 |
| Rank 1 → Channel | Channel Temperature Sensor |
| Rank 1 → Sampling Time | 640.5 Cycles |

先關 DMA Continuous Requests 再刪 DMA，是因為 CubeMX 在它為 Enabled 時要求必須有 Circular 的 DMA request，順序反了會跳警告。

### 步驟 4：檢查 NVIC

**System Core → NVIC**：**ADC1 and ADC2 interrupts** 要勾選，Preemption Priority **5**。（刪掉 DMA 後，DMA1 channel1 的中斷會自動消失，是正常的。）

### 步驟 5：新增 semaphore

**Middleware and Software Packs → FREERTOS → Timers and Semaphores** 分頁 → **Binary Semaphores** → **Add**：

- Semaphore Name：例如 `adcsem`（**程式碼裡的 handle 名稱 = 這個名字 + `Handle`**，你填 `adcsem` 就是 `adcsemHandle`，大小寫要一致）
- Allocation：Dynamic

產生的程式碼裡可以確認初始值：`osSemaphoreNew(1, 0, &adcsem_attributes);`，第二個參數 **0 = 初始為空**，正是我們要的（task 一開始會阻塞等中斷，不會讀到假資料）。

### 步驟 6：產生程式碼

**GENERATE CODE**（輸出到**同一個專案資料夾**）。

---

## 3. 【CubeIDE】修改程式碼

先在專案按右鍵 **Refresh（F5）**，讓 CubeIDE 讀到更新後的檔案。以下都寫在 `main.c` 的 `USER CODE` 區塊內。

### ① `USER CODE BEGIN PV`

```c
// 刪除（Example 版）：
//   uint16_t adcRaw[1];
//   volatile uint8_t adcDone = 0;

// 改成：
volatile uint16_t adcValue;     // ISR 讀到的轉換結果，交給 task 用
```

### ② `main()` 的 `USER CODE BEGIN 2`：整段清空

```c
/* USER CODE BEGIN 2 */
/* USER CODE END 2 */
```

（Example 版的校準、`HAL_ADC_Start_DMA`、`HAL_TIM_Base_Start(&htim2)` 全部刪掉，改到 task 裡。）

### ③ `USER CODE BEGIN 4`：callback 換掉

```c
int _write(int file, char *ptr, int len)       // 保留，讓 printf 輸出到 UART1
{
    HAL_UART_Transmit(&huart1, (uint8_t*)ptr, len, HAL_MAX_DELAY);
    return len;
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
    if (hadc->Instance == ADC1)
    {
        adcValue = HAL_ADC_GetValue(hadc);     // 讀出結果（同時清除 EOC 旗標）
        osSemaphoreRelease(adcsemHandle);      // 通知 task：有新資料了（名稱要和 CubeMX 一致）
    }
}
```

### ④ `StartDefaultTask` 的 `USER CODE BEGIN 5`：整段換掉

```c
HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED);   // 1. 校準（必須在啟動 ADC 之前）
HAL_ADC_Start_IT(&hadc1);                                // 2. 啟動 ADC（中斷模式），等 TIM1 觸發
HAL_TIM_Base_Start(&htim1);                              // 3. 啟動 TIM1，開始定頻觸發（注意是 htim1）

for(;;)
{
    osSemaphoreAcquire(adcsemHandle, osWaitForever);     // 睡到中斷來叫醒
    uint16_t raw = adcValue;
    int32_t temp = __HAL_ADC_CALC_TEMPERATURE(3300, raw, ADC_RESOLUTION_12B);
    printf("raw = %u, temp = %ld C\r\n", raw, (long)temp);
}
```

### 為什麼啟動動作移到 task 裡

`USER CODE BEGIN 2` 在 `osKernelInitialize()` 之前，而 semaphore 是在那之後才建立的。如果 TIM1 先啟動、第一次觸發恰好在 semaphore 建立前發生，callback 會拿到還沒建立的 `adcsemHandle`。放在 task 裡可以**保證 semaphore 一定已存在**。

### 資料流（Problem 1）

```
TIM1 每 100 ms Update Event
   ↓ TRGO（晶片內部連線）
ADC1 轉換內部溫度感測器（單次）
   ↓ EOC 中斷
HAL_ADC_ConvCpltCallback（ISR 裡：讀值 + osSemaphoreRelease）
   ↓
task 被喚醒（沒有輪詢延遲）→ 換算 → printf → UART
```

---

## 4. 編譯、燒錄、驗證

1. **Build**，確認 0 errors
2. **Run** 燒錄；序列終端機連線 115200 8N1（方法見實作筆記第 5 節）
3. 預期：**每秒約 10 行**，溫度約 20～45 °C

**頻率驗證**：用手機碼表量 10 秒，應有約 **100 行**。（光看「跑很快」不能確定是 10 Hz，建議實際量一次。）

**選用的視覺化驗證**：在 callback 裡暫時加 `HAL_GPIO_TogglePin(LED2_GPIO_Port, LED2_Pin);`，LED2（綠色）會閃爍，亮暗一個週期 = 兩次取樣；驗證完刪掉。

**實測輸出範例**（本專案）：

```
raw = 946, temp = 29 C
raw = 945, temp = 29 C
raw = 944, temp = 28 C
```

- **raw** 是 ADC 轉換出來的原始數字（0～4095），不是溫度；溫度是由 `__HAL_ADC_CALC_TEMPERATURE` 從 raw 換算出來的。
- raw 在 944～947 間跳動是 ADC 雜訊（12-bit 最後一兩個位元會被雜訊影響）；要降低可以用過取樣平均。
- temp 偶爾 29 變 28，是因為 `temp` 是整數，小數被捨去，raw 剛好落在整數邊界；不是溫度真的跳 1 °C。

---

## 5. 這次改的過程中踩過的坑

| 現象 | 原因 / 處理 |
|---|---|
| 編譯錯誤：`adcRaw`、`htim2` 未宣告 | `USER CODE BEGIN 2` 還留著 Example 版的程式，整段刪掉 |
| 編譯錯誤：`adcSemHandle` 未宣告 | CubeMX 產生的名稱是你填的 semaphore 名稱 + `Handle`（填 `adcsem` 就是 `adcsemHandle`），程式裡大小寫要完全一致 |
| 完全沒輸出 | 優先檢查：ADC 觸發來源是不是 **Timer 1 Trigger Out event**（不是 Out 2）；TRGO 是不是 Update Event；`HAL_TIM_Base_Start` 是不是 `&htim1`；`stm32l4xx_hal_msp.c` 的 `HAL_ADC_MspInit` 裡還有 `HAL_NVIC_EnableIRQ(ADC1_2_IRQn)`（ADC 中斷沒開，callback 永遠不會被呼叫） |
| 輸出狂刷、不是 10 Hz | Continuous Conversion Mode 沒設 Disabled（連續模式下 Timer 只觸發第一次） |
| 重新產生程式碼後 CubeIDE 沒更新 | 專案按右鍵 Refresh（F5） |
| `MX_GPIO_Init` 裡還有 `ARD_D4_Pin` 設成 `GPIO_AF1_TIM2` | TIM2 留下的腳位設定，TIM2 已不使用，不影響功能；想清乾淨就在 Pinout 圖點 PA3 選 Reset_State（選用） |
| 開機立刻多印一行怪數值 | semaphore 初始值若是 1 會這樣；檢查產生的 `osSemaphoreNew(1, 0, ...)` 第二個參數應為 0 |

---

## 6. 向助教說明參數（重點版）

**一句話版**：TIM1 時脈 80 MHz，PSC = 8000-1、ARR = 1000-1，得到 10 Hz；每次 Update Event 經 TRGO 直接觸發 ADC1，轉換完成產生 EOC 中斷，ISR 只讀值並 release semaphore，RTOS task 被喚醒後用 UART 印出。

| 參數 | 設定 | 理由 |
|---|---|---|
| Timer 時脈 | 80 MHz | MSI 4 MHz，PLL ×40 ÷2；AHB、APB2 預除頻皆為 1；這塊板子沒焊 HSE 晶振（UM2153 第 19 頁），所以不用外部晶振 |
| PSC / ARR | 8000-1 / 1000-1 | TIM1 是 16 位元，80 MHz ÷ 10 Hz = 8,000,000 超過 65535，必須拆成兩個欄位相乘 |
| 為什麼 10 Hz | | ① 遠大於一次轉換時間（約 13.6 µs，約 7000 倍）② 大於 UART 印一行的時間（約 2 ms，約 50 倍）③ 溫度變化很慢，足夠 |
| 為什麼用 Timer 而不是 osDelay | | 硬體計時，不受 RTOS 排程影響，沒有 drift、jitter 小 |
| Continuous = Disabled | | 連續模式下外部觸發只啟動第一次，之後 ADC 自己連續轉換，無法固定頻率（RM0351 ADC 章節與 ST HAL 說明皆有說明） |
| Sampling Time 640.5 cycles | | 溫度感測器輸出阻抗高，需要較長取樣時間讓保持電容充飽；ADC 時脈約 48 MHz，約 13.3 µs |
| 不用 DMA | | 題目要求每次轉換完成就產生中斷；DMA 是 Problem 2 |
| NVIC 優先權 5 | | FreeRTOS 規定會呼叫 RTOS API 的中斷，優先權數值不能小於 `configMAX_SYSCALL_INTERRUPT_PRIORITY`（預設 5） |
| semaphore（初始 0） | | task 阻塞等待，不佔 CPU，中斷一來立即喚醒；初始為空，避免開機讀到假資料 |
| ISR 只設通知 | | ISR 會搶佔所有 task，UART 印字慢（一行約 2 ms），放在 task 裡 |
| SYS Timebase = TIM6 | | SysTick 歸 FreeRTOS；TIM1 要留給 ADC 觸發 |

**請自己查證再回答的項目**（我是憑印象，沒有驗證）：
- 內部溫度感測器在 datasheet 的**最小取樣時間**（我印象中約 5 µs），確認 640.5 cycles（約 13.3 µs）有滿足
- `__HAL_ADC_CALC_TEMPERATURE` 的實作原理（我印象中是用出廠校準值線性內插，並假設供電 3300 mV）
- 高頻取樣（例如 1 kHz）會有什麼問題：間隔比印一行的時間短，binary semaphore 不累積計數，印字期間的通知會被合併而遺失資料，這時要改用 DMA + buffer（Problem 2）

---

## 7. 小提醒

- 回 CubeMX 改設定、重新產生程式碼之前，確認你的程式碼都寫在 `USER CODE BEGIN … END` 之間。
- `.ioc` 一定要開**專案資料夾裡的那一份**，否則產生的程式碼和你在 CubeIDE 看到的專案對不上。
- 想把頻率改成別的值，只要改 PSC 和 ARR（兩者都不能超過 65535）；改完要同步更新向助教說明的數字。
