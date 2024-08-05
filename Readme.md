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


在開始學習 RTOS 作業系統前，需要先了其解如何運作，如下圖所示為基本運作圖，展示了各 Task 如何交互與之間的關係

![FreeRTOS-TaskWork](/images/FreeRTOS-TaskWork.png)



## 資料型態與命名規則

