# FreeRTOS on Cortex-M

#advanced #rtos #arm #cortex-m #freertos

> FreeRTOS is the most popular embedded RTOS in the world. On Cortex-M, it uses PendSV, SysTick, and assembly-level stack manipulation to give the illusion of simultaneous task execution.

---

## 📖 What FreeRTOS Provides

FreeRTOS gives you:
- **Tasks**: Independent threads, each with its own stack
- **Scheduler**: Preemptive, priority-based (or cooperative)
- **Synchronization**: Semaphores, mutexes, event groups
- **Communication**: Queues, stream buffers, message buffers
- **Timers**: Software timers (non-interrupt context)
- **Memory**: 5 heap allocation schemes (heap_1 to heap_5)

---

## 🔬 FreeRTOS Architecture on Cortex-M

```
┌─────────────────────────────────────────────────────┐
│                   Application Tasks                  │
│   Task A (prio 1) │ Task B (prio 2) │ Task C (prio 1)│
└────────────────────┼─────────────────┼───────────────┘
                     │                 │
          ┌──────────▼─────────────────▼──────────┐
          │          FreeRTOS Kernel               │
          │  Scheduler │ Queues │ Semaphores       │
          │  ──────────────────────────────────    │
          │  SysTick ISR (tick count, preemption)  │
          │  PendSV ISR (context switch)            │
          │  SVC ISR (kernel entry, optional)      │
          └──────────────────────────────────────┘
                             │
          ┌──────────────────▼──────────────────┐
          │         Cortex-M Hardware            │
          │  NVIC │ SysTick │ PendSV │ MPU       │
          └─────────────────────────────────────┘
```

---

## 💻 Code Examples

### Minimal FreeRTOS Setup on STM32F407

```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"

/* Task handles (optional — needed for suspend/resume/delete) */
TaskHandle_t xLEDTask   = NULL;
TaskHandle_t xUARTTask  = NULL;

/* Shared queue */
QueueHandle_t xDataQueue = NULL;

/* ─── Task Definitions ─── */

void vLEDTask(void *pvParameters) {
    /* Setup: runs once when task starts */
    GPIO_Init_LED();
    
    TickType_t xLastWakeTime = xTaskGetTickCount();
    
    for(;;) {   /* Task must never return */
        LED_Toggle();
        
        /* Non-blocking delay — FreeRTOS delays suspend this task
         * and run others. Unlike busy-wait which blocks the CPU! */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(500));
        /* Wakes exactly 500ms after last wake — period-accurate! */
    }
}

void vUARTTask(void *pvParameters) {
    uint8_t received_byte;
    
    for(;;) {
        /* Block waiting for data from queue — consumes zero CPU while waiting */
        if (xQueueReceive(xDataQueue, &received_byte, portMAX_DELAY) == pdPASS) {
            UART_SendByte(received_byte);
        }
    }
}

/* ─── Main ─── */

int main(void) {
    /* Hardware init */
    SystemInit();   /* Configure clocks */
    UART_Init(115200);
    
    /* Create shared queue (10 bytes, 1 byte per item) */
    xDataQueue = xQueueCreate(10, sizeof(uint8_t));
    configASSERT(xDataQueue != NULL);
    
    /* Create tasks
     * xTaskCreate(function, name, stack_words, param, priority, handle) */
    xTaskCreate(vLEDTask,  "LED",  128, NULL, 1, &xLEDTask);
    xTaskCreate(vUARTTask, "UART", 256, NULL, 2, &xUARTTask);
    /*                                            ↑ priority 2 > 1: UART preempts LED */
    
    /* Start scheduler — never returns */
    vTaskStartScheduler();
    
    /* Should NEVER reach here — indicates heap_too_small if it does */
    configASSERT(0);
}
```

### Key FreeRTOSConfig.h Settings for Cortex-M

```c
/* FreeRTOSConfig.h — must be tailored to your chip */

#define configCPU_CLOCK_HZ              168000000UL  /* STM32F407 @ 168 MHz */
#define configTICK_RATE_HZ              1000         /* 1ms tick = 1kHz */
#define configMAX_PRIORITIES            5            /* 0-4 priority levels */
#define configMINIMAL_STACK_SIZE        128          /* Minimum task stack in words */
#define configTOTAL_HEAP_SIZE           (32 * 1024)  /* 32KB total FreeRTOS heap */

#define configUSE_PREEMPTION            1   /* Preemptive scheduling */
#define configUSE_IDLE_HOOK             0   /* No idle hook */
#define configUSE_TICK_HOOK             0

/* CRITICAL for FreeRTOS on Cortex-M: */
/* Must match NVIC priority grouping! 
 * If NVIC uses 4 priority bits (STM32 default): 
 * configMAX_SYSCALL_INTERRUPT_PRIORITY = 0x50 (decimal 80)
 * = priority 5 in 4-bit space (highest priority that can call FromISR APIs) */
#define configKERNEL_INTERRUPT_PRIORITY         255  /* Lowest priority = 0xFF */
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    80   /* 0x50 — priority 5 */

/* Debug: enable these during development */
#define configCHECK_FOR_STACK_OVERFLOW    2   /* Stack overflow hook */
#define configUSE_MALLOC_FAILED_HOOK      1   /* malloc failed hook */
#define INCLUDE_vTaskDelete               1
#define INCLUDE_vTaskSuspend              1
#define INCLUDE_uxTaskGetStackHighWaterMark 1

/* Map FreeRTOS interrupt handlers to correct names for STM32 */
#define vPortSVCHandler    SVC_Handler
#define xPortPendSVHandler PendSV_Handler
#define xPortSysTickHandler SysTick_Handler
```

### Task Synchronization — Queue from ISR

```c
/* Sending data from a UART ISR to a task (safe with FreeRTOS) */

QueueHandle_t xUartQueue;

void USART1_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    if (USART1->SR & USART_SR_RXNE) {
        uint8_t received = (uint8_t)(USART1->DR & 0xFF);
        
        /* Send from ISR — NEVER use xQueueSend() from ISR! */
        xQueueSendFromISR(xUartQueue, &received, &xHigherPriorityTaskWoken);
    }
    
    /* If sending unblocked a higher-priority task, yield to it immediately */
    /* This is what makes FreeRTOS responsive — no extra delay */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### Mutex for Shared Resource Protection

```c
SemaphoreHandle_t xI2CMutex = NULL;

void task_sensor_A(void *pvParam) {
    for(;;) {
        /* Take mutex — wait up to 100ms */
        if (xSemaphoreTake(xI2CMutex, pdMS_TO_TICKS(100)) == pdPASS) {
            /* We have exclusive I2C access */
            I2C_WriteRegister(0x48, REG_CONFIG, 0x01);
            uint16_t temp = I2C_ReadRegister(0x48, REG_TEMP);
            
            xSemaphoreGive(xI2CMutex);  /* Release ASAP */
            
            process_temperature(temp);
        } else {
            /* Timeout — I2C bus busy, handle error */
            i2c_timeout_count++;
        }
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

### Stack Overflow Hook

```c
/* Called by FreeRTOS when stack overflow detected (configCHECK_FOR_STACK_OVERFLOW=2) */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    /* DO NOT use any FreeRTOS APIs here — the stack is corrupted! */
    /* Just capture info and halt */
    volatile char *name = pcTaskName;  /* Read before stack corruption spreads */
    (void)name;
    
    /* Log to shared memory if available */
    /* In production: save to Flash, reset */
    __asm volatile("bkpt #0");  /* Break into debugger if attached */
    while(1);
}

/* Called when heap allocation fails */
void vApplicationMallocFailedHook(void) {
    /* Heap is full — increase configTOTAL_HEAP_SIZE */
    configASSERT(0);
}
```

---

## 🌍 Real-World: Sizing Tasks Correctly

```bash
# After running your application for several minutes:
# Check how much stack each task actually used (watermark = minimum free)

(gdb) print uxTaskGetStackHighWaterMark(xLEDTask)
# Returns: 84 words minimum free
# Created with 128 words → actually uses 128-84 = 44 words peak

(gdb) print uxTaskGetStackHighWaterMark(xUARTTask)
# Returns: 12 words minimum free
# DANGER! Very close to overflow → increase to 512 words
```

**Rule of thumb for stack sizing:**
- Idle task: 128 words (configMINIMAL_STACK_SIZE)
- Simple GPIO task: 128-256 words
- UART/SPI task with buffer: 256-512 words
- Task using printf: 512+ words (printf uses ~200 words of stack!)
- Task calling user functions: add function's stack depth × 2 safety margin

---

## ⚠️ Common Pitfalls

- **Calling non-FromISR APIs in ISR**: `xQueueSend()` in ISR → immediate HardFault or deadlock. ALWAYS use `xQueueSendFromISR()` in interrupt context.
- **configMAX_SYSCALL_INTERRUPT_PRIORITY wrong**: If set too low, kernel APIs mask your critical ISRs. If set too high, an ISR above the threshold calls a FreeRTOS API → kernel corruption. Know your priority mapping!
- **Forgetting `portYIELD_FROM_ISR`**: If you send to a queue from ISR but don't call `portYIELD_FROM_ISR`, the unblocked high-priority task doesn't run immediately — adding latency.
- **Task stack in BSS without zero-init**: FreeRTOS allocates task stacks from its heap (already zeroed). If you use static stack arrays, ensure they're zero-initialized.
- **Heap too small**: `vTaskStartScheduler()` silently returns if heap allocation fails. `configASSERT(0)` after it catches this.

---

## 🔗 Related Notes

- [[../Context_Switching/Context_Switch_Mechanism|Context Switch Mechanism]]
- [[../Task_Scheduling/Scheduling_Algorithms|Scheduling Algorithms]]
- [[../Synchronization/Mutex_and_Semaphores|Mutex and Semaphores]]
- [[../../05_Cortex_M_Deep_Dive/NVIC/NVIC_Deep_Dive|NVIC Deep Dive]]
- [[../../05_Cortex_M_Deep_Dive/SysTick/SysTick_Timer|SysTick Timer]]
- [[../../12_Projects/04_RTOS_Scheduler/RTOS_Scheduler_Project|Project 4: Mini RTOS Scheduler]]

---

## 🎯 Interview Questions

1. **What FreeRTOS interrupt handlers map to Cortex-M exceptions?**
   *SVC_Handler → FreeRTOS kernel entry; PendSV_Handler → context switching; SysTick_Handler → tick count and preemption trigger.*

2. **What is `configMAX_SYSCALL_INTERRUPT_PRIORITY` and why does it matter?**
   *The highest interrupt priority that is allowed to call FreeRTOS API functions. ISRs above this priority level cannot call FreeRTOS APIs (they're not masked by kernel critical sections). Setting this wrong causes kernel corruption or locks out critical ISRs.*

3. **Why is `vTaskDelay()` better than a busy-wait loop in a task?**
   *`vTaskDelay()` puts the task in Blocked state, allowing the scheduler to run other tasks. A busy-wait loop keeps the task in Running state, consuming CPU and starving lower-priority tasks.*

4. **What's the difference between a mutex and a binary semaphore in FreeRTOS?**
   *A mutex includes priority inheritance — if a low-priority task holds it and a high-priority task is waiting, the low-priority task temporarily gets the higher priority to prevent priority inversion. A binary semaphore has no priority inheritance.*

---

*← [[../MOC_RTOS|Stage 8: RTOS on ARM]]*
