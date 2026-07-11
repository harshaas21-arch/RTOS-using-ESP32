# RTOS-using-ESP32
# Mastering FreeRTOS on ESP32: From Scratch to Concurrency

A comprehensive documentation repository detailing the foundational paradigms of Real-Time Operating Systems (RTOS) using the **Espressif ESP-IDF framework**. This documentation covers core execution mechanics, task memory abstraction, inter-task messaging networks, hardware mutual exclusion, and hardware-level asymmetric dual-core execution.

---

## 📂 Core Concepts & API Reference Guide

### 1. Fundamentals of Tasks & Delay Scheduling
In an RTOS workspace, application flows are broken down into independent execution loops called **Tasks**. Unlike sequential bare-metal scripts, tasks are scheduled dynamically by the kernel based on priority levels and timing requests.

#### 🛠️ Essential APIs Used
*   `xTaskCreate(...)`: Allocates stack memory and registers a custom loop with the system scheduler.
*   `vTaskDelay(...)`: Voluntarily suspends the calling task for a specific number of clock ticks, freeing the processor for other execution flows.
*   `pdMS_TO_TICKS(ms)` / `portTICK_PERIOD_MS`: Macros used to translate physical milliseconds into internal scheduler system time units.

#### 📋 Code Implementation
```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void print_message_task(void *pvParameters){
    while(1){
        printf("Learning RTOS from scratch \n");
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

void counter_task(void *pvParameters){
    int count = 0;
    while(1){
        printf("Counter: %d\n", count);
        count++;
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void app_main(void){
    xTaskCreate(print_message_task, "Print Task", 2048, NULL, 5, NULL);
    xTaskCreate(counter_task, "Counter Task", 2048, NULL, 5, NULL);
}
```

📺 Console Output Behavior
Learning RTOS from scratch 
Counter: 0
Counter: 1
Counter: 2
Counter: 3
Learning RTOS from scratch
Analysis: Because both tasks operate at identical priority metrics (5), the scheduler divides runtime evenly using round-robin scheduling. The counter runs exactly 4 times for every 1 print from the primary message task.

### 2. Dynamic Task Abstraction via Parameters (pvParameters)

Hardcoding unique logic tasks for identical targets reduces memory efficiency. FreeRTOS handles task instantiation reuse by utilizing a generic pointer configuration argument (void *pvParameters).

#### 🛠️ Mechanics Breakdown
A void * parameter acts as a universal reference mechanism in C. It allows data payloads (primitives, arrays, configurations, structs) to pass smoothly into identical function tasks.

📋 Code Implementation
```c
typedef struct {
    char *task_message;
    int delay_ms;
} Task_Config_t;

void generic_print_task(void *pvParameters){
    Task_Config_t *myconfig = (Task_Config_t *) pvParameters;
    while(1){
        printf("%s \n", myconfig->task_message);
        vTaskDelay(myconfig->delay_ms / portTICK_PERIOD_MS);
    }
}

Task_Config_t task1_data = { .task_message = "[Instance 1] Fast task", .delay_ms = 400};
Task_Config_t task2_data = { .task_message = "[Instance 2] Slow task", .delay_ms = 1500};

void app_main(void){
    xTaskCreate(generic_print_task, "Fast print", 2048, &task1_data, 5, NULL);
    xTaskCreate(generic_print_task, "Slow print", 2048, &task2_data, 5, NULL);
}
```
### 3. Inter-Task Thread-Safe Communication via Queues
Directly sharing memory locations or global storage flags between running loops causes memory corruption or data race vulnerabilities. Queues provide a secure First-In, First-Out (FIFO) conveyor belt managed directly by the kernel core.

#### Function Name,Description,Key Parameters
"xQueueCreate(length, item_size)",Allocates RAM blocks for thread-safe buffer structures.,length: Max element limits.item_size: Storage element footprint in bytes.
"xQueueSend(queue, pvItemToQueue, xTicksToWait)",Safely appends a data record element to the end of a queue.,queue: Handle reference.pvItemToQueue: Variable pointer source.
"xQueueReceive(queue, pvBuffer, xTicksToWait)",Safely retrieves and drops an item from the front of the queue.,xTicksToWait: Use portMAX_DELAY to fully block and suspend the task until data arrives.

```c
#include "freertos/queue.h"

QueueHandle_t my_first_queue;

void producer_task(void *pvParameters){
    int value_to_send = 0;
    while(1){
        value_to_send++;
        printf("[Producer] Created data: %d \n", value_to_send);
        xQueueSend(my_first_queue, &value_to_send, 0);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
}

void consumer_task(void *pvParameters){
    int received_value = 0;
    while(1){
        if(xQueueReceive(my_first_queue, &received_value, portMAX_DELAY) == pdPASS){
            printf("[Consumer] Safely grabbed data: %d \n", received_value);
        }
    }
}
```
### 4. Mutual Exclusion and Resource Management via Mutexes
When two processing routines target a shared peripheral or memory interface concurrently (such as an LCD display or a UART console), the data output streams can become garbled. A Mutex acts as an atomic token lock guaranteeing exclusive access.

📋 The Conflict (No Lock Control)
Without lock implementations, task threads cut in midway through data output processes:

[Task Alpha] Starting transmission 
[Task Beta] Starting transmission 
Status is: Running 
Status is: Critical_Error

🛠️ The Solution (Mutex Integration APIs)
xSemaphoreCreateMutex(): Spawns an atomic binary lock engine.

xSemaphoreTake(mutex, timeout): Obtains exclusive ownership rights. If the key is checked out, the requesting thread halts.

xSemaphoreGive(mutex): Returns the token, unblocking subsequent waiting threads.

```c
#include "freertos/semphr.h"
SemaphoreHandle_t serial_mutex;

void send_secure_message(const char *name, const char *status){
    if(xSemaphoreTake(serial_mutex, portMAX_DELAY) == pdTRUE){
        printf("[%s] Starting transmission \n", name);
        vTaskDelay(10 / portTICK_PERIOD_MS); // Simulates peripheral transmission wait times
        printf("Status is: %s \n", status);
        xSemaphoreGive(serial_mutex);
    }
}
```

### 5. Asynchronous Hardware Parallelism via Dual-Core Pinning
The ESP32 architecture contains two distinct physical processing units (Core 0 and Core 1). While standard tools manage routing abstractly, developers can pin computational loads manually to specific cores to achieve true hardware parallelism.

🛠️ Extended Dual-Core API
xTaskCreatePinnedToCore(...): Accepts a 7th argument (BaseType_t xCoreID) to map the processing path directly to an isolated piece of silicon.

xPortGetCoreID(): Checks the execution context at runtime, returning 0 or 1.

```c
void core_zero_worker(void *pvParameters) {
    while(1){
        printf("[Task 1] Running on Core ID : %d \n", xPortGetCoreID());
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
}

void core_one_worker(void *pvParameters){
    while(1){
        printf("[Task 2] Running on Core ID : %d \n", xPortGetCoreID());
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
}

void app_main(void){
    xTaskCreatePinnedToCore(core_zero_worker, "Core 0 Task", 2048, NULL, 5, NULL, 0);
    xTaskCreatePinnedToCore(core_one_worker, "Core 1 Task", 2048, NULL, 5, NULL, 1);
}
```
Parallel Output Execution
Spawning Dual core framework.. 
[Task 1] Running on Core ID : 0 
[Task 2] Running on Core ID : 1 
[Task 1] Running on Core ID : 0 
[Task 2] Running on Core ID : 1

---

### 6. Task Signaling via Binary Semaphores
Unlike Mutexes (which handle mutual exclusion and feature priority inheritance), a **Binary Semaphore** is used for **Task Synchronization and Signaling**. It tracks state change notifications rather than ownership locks. Any task or Interrupt Service Routine (ISR) can "Give" a semaphore to unblock a waiting task.

#### 🛠️ Core APIs
*   `vSemaphoreCreateBinary(xSemaphore)` or `xSemaphoreCreateBinary()`: Initializes a binary flag signaling engine.
*   `xSemaphoreTake(xSemaphore, xBlockTime)`: Halts a task until the signal token is available.
*   `xSemaphoreGive(xSemaphore)`: Releases the signaling token to wake up any blocked tasks.

---

### 7. Low-Overhead Periodic Work via Software Timers
Creating fully dedicated execution threads for simple recurring tasks (like low-frequency sensor checking or timeout routines) consumes excessive RAM due to task stack overhead. **Software Timers** allow code execution at strict intervals without allocating individual task blocks, relying instead on a centralized system `Timer Daemon Task`.

#### 🛠️ Essential APIs Used
*   `xTimerCreate(...)`: Creates a dynamic tracking timer container configuration.
    *   *Auto-Reload flag*: Set to `pdTRUE` for persistent periodic recurrence; set to `pdFALSE` for single "One-Shot" execution.
*   `xTimerStart(...)`: Pushes a start command onto the internal timer management queue.

#### 📋 Code Implementation
```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/timers.h"

TimerHandle_t my_periodic_timer;

void timer_callback(TimerHandle_t xTimer) {
    printf("[Software Timer] Callback fired! Running without a dedicated task thread.\n");
}

void app_main(void) {
    printf("Initializing Software Timers ....\n");

    my_periodic_timer = xTimerCreate(
        "Periodic Timer",
        (1500 / portTICK_PERIOD_MS),
        pdTRUE, // Auto-Reload Enabled
        (void *) 0,
        timer_callback
    );

    if (my_periodic_timer != NULL) {
        xTimerStart(my_periodic_timer, 0);
    }
}
```

---

### 8. Multi-Event Coordination via Event Groups
When an execution routine requires **multiple separate conditions to be met simultaneously** before unblocking (such as waiting for multiple hardware modules to finish booting), using multiple nested semaphores introduces code complexity. An **Event Group** provides an atomic bitmask where a task can remain suspended until an exact pattern of bits (`AND` or `OR` combinations) is set.

#### 🛠️ Essential APIs Used
*   `xEventGroupCreate()`: Spawns the internal event tracking bitmask workspace.
*   `xEventGroupSetBits(handle, bits_to_set)`: Atomically transitions specific bit positions high.
*   `xEventGroupWaitBits(...)`: Halts a task until target bit flags are cleared or filled.

---

### 9. High-Speed, Zero-RAM Signaling via Task Notifications
Creating distinct synchronization primitives (like queues or binary semaphores) incurs both RAM footprint and scheduler lookup overhead. **Task Notifications** allow threads to signal each other directly by writing straight to a 32-bit tracking register pre-allocated within the destination task's **Task Control Block (TCB)**. This bypasses object handles entirely, yielding a performance improvement of roughly **45%**.

#### 📋 Code Implementation
```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

TaskHandle_t receiver_task_handle = NULL;

void sender_task(void *pvParameters) {
    while(1) {
        vTaskDelay(2500 / portTICK_PERIOD_MS);
        printf("[Sender Task] Work complete. Injecting direct notification...\n");
        xTaskNotifyGive(receiver_task_handle); // Target the task handle directly
    }
}

void receiver_task(void *pvParameters) {
    while(1) {
        printf("[Receiver Task] Going to sleep, waiting for a direct notification...\n");
        uint32_t notification_value = ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        printf("[Receiver Task] Woke up! Token event count: %d\n", (int)notification_value);
    }
}

void app_main(void) {
    xTaskCreate(receiver_task, "Receiver", 2048, NULL, 5, &receiver_task_handle);
    xTaskCreate(sender_task, "Sender", 2048, NULL, 5, NULL);
}
```
📺 Console Output Behavior

Initializing Software Timers ....
I (254) main_task: Returned from app_main()
[Software Timer] Callback fired! Running without a dedicated task thread.
[Software Timer] Callback fired! Running without a dedicated task thread.
