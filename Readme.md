# RTOS

以 Free RTOS open source 實作，搭配 STM32 與 HT32 系列開發版

</br>

# 簡介與基礎認知

### 前後台系統

前後台系統常建設在一般的程序上，簡單來說就是直接裸機操作 (直接套用while無限回圈內)。
須注意此系統並沒有嵌入式操作系統的改念

* 前台系統：中斷 (interrupt) 用於處理系統的異步事件 (callback)。

* 後台系統：程序是個死循環，在循環中不斷調用 API 函數完成所需的事件。

```
NOTE !! ：

前台系統為中斷級，後台為任務級
```

</br>

![Front desk system diagram](/images/Front_desk_system_diagram.png)

</br>


### RTOS 系統

即為即時性作業系統，並強調即時性，又分為軟即時與硬即時。將需要實現的功能劃分為多個任務，並具有可剝奪性。

代表作：FreeRTOS、UCOS、RT-Thread、DJYOS

* 軟即時：系統能讓絕大多數任務在確定時間內完成。

* 硬即時：系統必須使任務在確定的時間內完成。

* 可剝奪性：CPU執行多個任務中優先權最高的那個任務，即使CPU正在執行某個低階任務，當高階任務準備好時，高階任務就會優先搶奪執行權。

</br>

![Backend system diagram](/images/Backend_system_digram.png)

</br>

# Free RTOS

## 簡介

FreeRTOS 是一個可剪裁、可剝奪型的多任務核心，而且沒有任務數量限制。

官網：https://www.freertos.org/index.html


## 原始碼

下載：https://freertos.org/a00104.html

* 核心程式碼：(Source)
    * tasks.c：掌管所有 task 的檔案
    * queue.c：管理 task 間的通訊
    * list.c：提供系統與應用實作會用到的 list 結構

* 硬體相關檔案：以 ARM Cortext-M3 為例，可在 Source/portable/GCC/ARM_CM3 中找到
    * portmacro.h：定義了與硬體相關的變數，如資料型態定義，以及與硬體相關的函式呼叫名稱定義(以 portXXXXX 命名)等，統一各平臺的函式呼叫
    * port.c：定義了包含與硬體相關的程式碼實作
    * FreeRTOSConfig.h：包含 clock speed, heap size, mutexes 等等都在此定義(需自行建立)

</br>

### 架構
FreeRTOS 的程式碼可以分為三個主要區塊：任務、通訊和硬體界面。

* 任務(Tasks)：FreeRTOS 的核心程式碼約有一半是用來處理多數作業系統首要關注的問題：任務，任務是擁有優先權的用戶所定義的 C 函數。task.c 和 task.h 負責所有關於建立、排程和維護任務的繁重工作。

* 通訊(Communication)：FreeRTOS 核心程式碼大約有 40% 是用來處理通訊的。queue.c 和 queue.h 負責處理 FreeRTOS 的通訊，任務和中斷(interrupt)使用佇列(佇列，queue)互相發送數據，並且使用 semaphore 和 mutex 來派發 critical section 的使用信號。(~~資料結構很重要~~)

* 硬體介面：同一份程式碼在不同硬體平台上的 FreeRTOS 都可以運行。大約有 6% 的 FreeRTOS 核心代碼，在與硬體無關的 FreeRTOS 核心和與硬體相關的程式碼間扮演著墊片(shim)的角色。

</br>
</br>

## 概念

在開始學習 RTOS 作業系統前，需要先了其解如何運作，如下圖所示為基本運作圖，展示了各 Task 如何交互與之間的關係

![FreeRTOS-TaskWork](/images/FreeRTOS-TaskWork.png)

1. 方格為狀態：
  * 運行：正在由 CPU 執行的狀態
  * 就緒：準備要執行的下一個狀態
  * 阻塞：在等待某個事件的狀態
  * 掛起：透過程式 API 要求退出之狀態

2. 方格之間的箭頭為需要透過地關係 (API)
  * Rusume：恢復狀態
  * Suspend：掛起狀態
  * Event (事件)：等待的事件
  * 阻塞函數：需要被等待的事件

</br>

經由第 2. 點可以發現，Rusume 與 Suspend 為相互的；Event 與 阻塞函數也為相互的。

</br>

接下來由流程的方式說明 RTOS 架構如何運作：

當一個任務被創建時 (視為黑點)，會處於就緒狀態。

任務被創建之後，使用者可以選擇要進行的動作，有兩種：運行、掛起；首先運行，就是直接運行該運務之內容；最後，掛起就是先將任務放置在一旁不進行運行，直到使用者需要時再透過 API 呼叫回復運行狀態。

其中較為特別的是阻塞，透過 API 的呼叫使任務被阻塞使得其他任務可以執行，這有關乎 RTOS 運作之關係， RTOS 之重點就在這，透過多個任務不斷互相阻塞與運行狀態達成類多執行序的效果。

</br>

---

</br>

以上說明可能較為難懂，以下由比喻的方式說明：

前情提要：
* 假設你的大腦是一個 MCU ，首先，你創建了三個任務分別是：動左腳、動右腳與躲避石頭。
* 當前情境為，你需要動左右腳進行走路，並且可能隨時會有人朝你丟石頭，所以你需要躲避。

</br>

各任務內容：
* 動左腳：抬起左腳 >> 放下左腳 >> 阻塞
* 動右腳：抬起右腳 >> 放下右腳 >> 阻塞
* 躲避石頭：躲避 >> 掛起
* 優先級：躲避石頭 > 動左腳 > 動右腳

</br>

開始執行：
* 創建各任務，其中躲避石頭創建完成後先掛起，因為躲避的動作不用時時刻刻進行；初始化完成，首先執行優先權級最大之任務，動左腳，當抬起左腳執行完成後，執行下一步驟放下左腳，此時進入阻塞狀態，系統開始尋找次優先權級之任務，動右腳，執行內部動作，執行完成後，再次尋找下一優先級，在這樣不斷得循環之中形成走路之動作。
  
* 突然有人朝你丟了石頭，你的大腦接收到了需要將躲避石頭恢復執行之動作，於是 Resume 了躲避石頭的任務，並將其執行，在執行完躲避的程序後，再將該任務掛起，因為並不需要一直處於躲避的狀態，直到下次需要應對時在恢復執行就好。

</br>

以上為基本的 RTOS 執行，實際上在撰寫與設計的過程中需要注意很多地方，例如：與中斷的交互、阻塞的等待事件、掛起與恢復執行的時間點...等等。

</br>

---

</br>

## 資料型態與命名規則

Free RTOS 不只是簡單的軟體流程而已，他還與硬體設備有關。

在不同的硬體設備上，通訊埠設定也會不同，其設定會在一個 portmarco.h 的標頭檔裡，有兩種特殊資料型態 portTickType 與 portBASE_TYPE。

* 資料型態：
  * portTickType：儲存 tick 計數值，可以用來判斷 block 次數。
  * portBASE_TYPE：定義為架構基礎的變數，隨各不同硬體來應用，如在 32Bit 架構上，其為 32Bit 型態，最常用以儲存極限值或布林數。

* 函式
  * 以回傳值型態與所在檔案名稱為開頭
    * vTaskPriority() 是 task.c 中回傳值型態為 **void** 的函式
    * xQueueReceive() 是 queue.c 中回傳值型態為 **portBASE_TYPE** 的函式
  * 只能在該檔案中使用的 (scope is limited in file) 函式，以 prv 為開頭 (private)

* 巨集
  * 巨集在 Free RTOS 裡皆為大寫字母定義，名稱前小寫字母為巨集定義的地方
    * portMAX_DELAY : portable.h
    * configUSE_PREEMPTION : FreeRTOSConfig.h

</br>

# Kernel Developer

## 任務 (Task)

為 Free RTOS 中執行的基本單位，每一個任務都由 C 的函數組成。

當使用 RTOS 的即時應用程式可以結構化一組獨立的任務，也就是每創建一個任務都會是一個單獨的結構體。

在任何時間點，RTOS 中只能有一個任務在執行，而即時 RTOS 排程器負責決定這個任務應該是哪一個。

因此，當應用程式執行時，RTOS 排程器可能會重複啟動和停止每項任務 (將每項任務交換進來和交換出去)。

由於任務並不知道 RTOS 排程器如何運作，因此即時 RTOS 排程器有責任確保在任務交換進來時的處理器上下文（暫存器值、堆疊內容等）與交換出來時的處理器上下文完全相同。為了達到這個目的，每個任務都有自己的堆疊。當任務被換出時，執行上下文會被儲存在該任務的堆疊中，因此當同樣的任務被換回時也可以完全還原。**簡單來說，當進行到一半的任務的執行權需要交出給下一個任務時 RTOS 會確保被中斷的點再下次該任務換回來時可以持續執行。**

</br>

### States

如上面所提到的可以分為：
* 運行 (Running)
* 就緒 (Ready)
* 阻塞 (Blocked)
* 掛起 (Suspend)

以下詳細解說

</br>

**Running**

當任務實際正在執行時，稱為 Running 狀態。
它目前正在使用處理器。如果執行 RTOS 的處理器只有一個核心，那麼任何時候都只能有一個任務處於 Running 狀態。

---

</br>

**Ready**

就緒任務是指可以**即將**執行的任務 (它們未處於 Blocked 或 Suspend 狀態)，但目前尚未執行，因為優先順序相同或更高的其他任務已處於執行狀態。

也就是執行權目前被其他任務佔有中。

---

</br>

**Blocked**

如果任務目前正在等待時間事件 (與 os 有關的 delay) 或外部事件，則稱該任務處於阻塞狀態。

例如，如果任務呼叫 vTaskDelay()，它會阻塞，直到延遲期間結束為止，這是一種時間事件。

任務也可以阻塞來等待佇列、訊號觸發、事件群組、通知。

阻塞狀態中的任務通常會有「 timeout 」，timeout 後，即使任務等待的事件尚未發生，任務也會超時，並解除阻塞。

**處於阻塞狀態的任務不使用任何處理時間，且無法選擇進入執行狀態。**

---

</br>

**Suspend**

與處於 Blocked 狀態的任務一樣，處於 Suspended 狀態的任務也無法選擇進入 Running 狀態，但是處於 Suspended 狀態的任務沒有 timeout。相反，任務只有在分別透過 vTaskSuspend() 和 xTaskResume() ...等 API 呼叫**明確命令**進入或離開 Suspended 狀態時，才會進入或離開 Suspended 狀態。

---

</br>

#### How to make a TASK

需要先定義一個 C 的函數，然後再用 xTaskCreate() 這個 API 來建立一個 task，此 C 函數有幾個特點，它的返回值必須是 void，其中通常會有一個無限迴圈，所有關於這個 task 的工作都會在迴圈中進行，且函數不會有 return，FreeRTOS 不允許 task 自行結束 ( 使用 return 或執行到函數的最後一行 )。


**建立一個 Task 的任務內容為何：**

```c
void vATaskFunction( void *pvParameters )
{
    for( ;; )
    {
        /* Task application code here. */
    }

    /* Tasks must not attempt to return from their implementing
       function or otherwise exit. In newer FreeRTOS port
       attempting to do so will result in an configASSERT() being
       called if it is defined. If it is necessary for a task to
       exit then have the task call vTaskDelete( NULL ) to ensure
       its exit is clean. */
}
```

</br>

**創建一個 Task：**

```c
xTaskCreate( TaskFunction_t pvTaskCode,
             const char * const pcName,
             const configSTACK_DEPTH_TYPE uxStackDepth,
             void *pvParameters,
             UBaseType_t uxPriority,
             TaskHandle_t *pxCreatedTask);
```

* pvTaskCode：就是我們定義好用來建立 task 的 C 函數
* pcName：任意給定的 task name，這個名稱只被用來作識別，不會在 task 管理中被採用
* usStackDepth：堆疊的大小 (以 Byte 計算)
* pvParameters：要傳給 task 的參數陣列，也就是我們在 C 函數宣告的參數，可以為 NULL
* uxPriority：定義這個任務的優先權，在 FreeRTOS 中，0 最低，(configMAX_PRIORITIES – 1) 最高
* pxCreatedTask：handle，是一個被建立出來的 task 可以用到的識別符號

</br>

## 優先權 (Priority)

在 Free RTOS 中定義最高優先權的變數為 **configMAX_PRIORITIES** ， 位於 FreeRTOSConfig.h

每個任務都會有各自的優先權級，從 0 到 configMAX_PRIORITIES ，根據不同的優先權依序執行 Ready 中的 Task，其中，低優先順序數字表示低優先順序工作，閒置任務的優先順序為零

Free RTOS 的調度程序會確保處於 Ready 或 Running 狀態的任務，總會優先得到處理器 (CPU) 的時間，而不是優先順序較低但也處於就緒狀態的任務。換句話說，置於 Running 狀態的任務永遠是能夠執行的最高優先順序任務。

任何數量的任務都可以共用相同的優先順序。如果未定義 configUSE_TIME_SLICING，或者 configUSE_TIME_SLICING 設定為 1，則具有相同優先順序的就緒狀態任務將使用時間切片輪流排程方案來分享可用的處理時間。

</br>

## 排程 (Scheduling)

Free RTOS 的排程演算法涉及到核心數量，不同數量的核心會牽扯到多種不同記憶體位置、任務調度、互斥量 ... 等問題。

Free RTOS 排程演算法適用於單核心，非對稱多核心 (AMP) 和對稱多核心 (SMP) 的設定。

排程演算法是決定哪個 RTOS 任務應處於 Running 狀態的軟體例行程序。在任何特定時間，每個處理器核心只能有一個任務處於 Running 狀態。AMP 是指每個處理器核心都執行自己的 FreeRTOS 範例。SMP 是指有一個 FreeRTOS 範例在多個核心之間排程 RTOS 任務。

</br>

### 單核心

預設情況下，FreeRTOS 使用固定優先順序的優先排程，並對等優先順序 (equal priority tasks) 的任務進行循環時間切分。

Equal priority tasks：

* Fixed priority：固定優先順序，表示排程器不會永久改變任務的優先順序，雖然它可能會因為優先順序繼承而暫時提升任務的優先順序。

* Preemptive：搶佔執行，是指調度器永遠執行能夠執行的最高優先順序 RTOS 任務，不論任務何時能夠執行。 例如，如果中斷程序 (ISR) 改變了能夠執行的最高優先順序任務，則調度器會停止目前執行的較低優先順序任務，並啟動較高優先順序的任務 - **即使這發生在時間片內**。 在這種情況下，較低優先順序的工作稱為被較高優先順序的工作「搶先執行」。

* Round-robin：循環，是指共享優先順序的任務輪流進入「運行」狀態。

* Time sliced：時間片，表示調度器會在每次 tick 中斷時切換優先順序相同的任務。(tick 中斷是 RTOS 用來測量時間的週期性中斷）。

</br>

```
Using a prioritised preemptive scheduler - avoiding task starvation
```

永遠執行最高優先順序工作的後果是，高優先順序工作若從未進入 Blocked 或 Suspended 狀態，將永遠剝奪所有較低優先順序工作的執行時間。 

例如：如果高優先順序任務正在等待事件，它就不應該在循環中等待事件，因為輪詢讓它一直在執行，所以永遠不會處於阻塞或暫停狀態；相反，該任務應該進入 Blocked 狀態來等待事件，事件可以使用 FreeRTOS 多種任務間通訊和同步 function 之一傳送給任務。 

接收事件會自動將優先順序較高的任務從 Blocked 狀態移除，較低優先順序的任務會在較高優先順序的任務處於 Blocked 狀態時執行。

</br>

#### 設定 RTOS 排程 ：

* configUSE_PREEMPTION
  * 如果 configUSE_PREEMPTION 為 0，則會關閉搶佔權，只有當執行中的任務進入 Blocked 或 Suspend 狀態時才會發生上下文切換，執行中的任務會呼叫 taskYIELD() 或中斷程序 (ISR) 手動要求上下文切換時，才會發生上下文切換。

* configUSE_TIME_SLICING
  * 如果 configUSE_TIME_SLICING 為 0，則時間分割關閉，因此排程器不會在每次 tick 中斷時切換優先順序相同的工作。

</br>

### 非對稱多核心(AMP)

使用 FreeRTOS 的非對稱多核心 (AMP) 是指多核心裝置的每個核心都執行自己獨立的 Free RTOS。 

這些核心不需要擁有相同的架構，但如果 Free RTOS 需要彼此通訊，則必須共用一些記憶體。

每個核心都執行自己的 FreeRTOS，因此任何指定核心上的排程演算法都與上述單核心系統的排程演算法完全相同。 

可以使用串流或訊息緩衝區作為核心間的通訊，這樣一個核心上的任務就可以進入 Blocked 狀態，以等待從不同核心傳送來的資料或事件。

</br>

### 對稱多核心 (SMP)

使用 FreeRTOS 的對稱多核心 (SMP) 是指一個 FreeRTOS 範例在多個處理器核心之間調度 RTOS 裡的任務。由於只有一個 FreeRTOS 範例在執行，所以同一時間只能使用一個 FreeRTOS port，因此每個核心必須有相同的處理器架構，並共享相同的記憶體空間。

SMP 會導致在任何特定時間都有超過一個任務處於 Running 狀態 (每個核心有一個 Running 狀態的任務)。 這表示只有在沒有較高優先順序的任務可以執行時，較低優先順序的任務才會執行的假設不再成立。 

SMP 排程器如何選擇要在雙核微控制器上執行的任務，當一開始有一個高優先順序任務和兩個中優先順序任務都處於就緒狀態時，排程器需要選擇兩個任務，每個核心一個。

首先，高優先級任務是能夠執行的最高優先級任務，因此它會被選取用於第一個核心。剩下的兩個中優先級任務是能夠執行的最高優先級任務，因此會為第二個核心選擇一個。結果是高優先級和中優先級工作同時執行。

</br>

