# RTOS & Scheduling — Interview Questions

#interview #rtos #arm #advanced

---

## 🟡 Intermediate Level

**Q1: What is the difference between a hard real-time and soft real-time system?**

> **Hard real-time**: Missing a deadline causes system failure. The consequence is catastrophic (airbag that deploys 200ms late is useless). Scheduling must be deterministic and provably bounded.
>
> **Soft real-time**: Missing a deadline degrades quality but doesn't cause failure. A video player that drops frames is annoying, not dangerous.
>
> FreeRTOS can satisfy soft real-time and many hard real-time requirements, but formal verification (using Rate Monotonic Analysis etc.) is needed for safety-critical hard real-time.

---

**Q2: What is preemption and when does it happen in FreeRTOS?**

> Preemption is when the RTOS forcibly removes a running task and switches to a higher-priority task. It happens in FreeRTOS:
> 1. When a higher-priority task unblocks (e.g., `xQueueSendFromISR` releases a blocked task)
> 2. On every SysTick tick (if a same-or-higher priority task is ready)
>
> With `configUSE_PREEMPTION=1` and `configUSE_TIME_SLICING=1`, tasks at the same priority share CPU in round-robin slices (one tick each). With `configUSE_TIME_SLICING=0`, same-priority tasks only switch if they block.

---

**Q3: What is priority inversion? Give a real-world example.**

> Priority inversion occurs when a high-priority task is blocked waiting for a resource held by a low-priority task, and a medium-priority task preempts the low-priority task — preventing it from ever releasing the resource.
>
> **Classic example (Mars Pathfinder, 1997)**:
> - High-priority: bus management task
> - Medium-priority: science data collection task
> - Low-priority: weather sensor task
>
> Weather task held a shared mutex. Bus task needed it but blocked. Science task preempted weather task (it was medium priority, higher than weather). Bus task starved → watchdog triggered → system reset.
>
> **Fix**: Priority inheritance (mutex temporarily boosts weather task to bus task's priority).

---

**Q4: What is the difference between a semaphore and a mutex?**

> | Feature | Binary Semaphore | Mutex |
> |---------|-----------------|-------|
> | Owner | No owner concept | Owned by the task that took it |
> | Priority inheritance | No | Yes (in FreeRTOS recursive mutex) |
> | Use case | Signaling between tasks/ISR | Mutual exclusion of shared resource |
> | ISR friendly | Yes (xSemaphoreGiveFromISR) | No — mutex cannot be given from ISR |
>
> Use **semaphore** for signaling: "ISR signals task that data is ready."
> Use **mutex** for mutual exclusion: "Only one task can use SPI at a time."

---

**Q5: How does FreeRTOS determine which task to run next?**

> FreeRTOS maintains a ready list for each priority level. `vTaskSwitchContext()` selects the highest-priority non-empty ready list and picks the task at the head. If two tasks share the highest priority, they alternate each tick (round-robin). The scheduler runs:
> 1. On each SysTick (preemption check)
> 2. When a task blocks (immediate reschedule)
> 3. When a higher-priority task is unblocked from an ISR (via `portYIELD_FROM_ISR`)

---

## 🔴 Advanced Level

**Q6: Walk me through what happens when a FreeRTOS task calls `vTaskDelay(100)`.**

> 1. Task calls `vTaskDelay(100)` (100 ticks = 100ms at 1kHz tick)
> 2. FreeRTOS calculates wake tick: `current_tick + 100`
> 3. Task is removed from the ready list and placed in the delayed list (sorted by wake time)
> 4. `vTaskSwitchContext()` is called → selects next ready task
> 5. PendSV fires → context switches to the next task
> 6. 100 SysTick ticks pass → delayed list is checked each tick
> 7. At tick 100, the task is moved back to the ready list
> 8. If it's now the highest-priority ready task, it preempts the current task on the next tick (or immediately via `portYIELD`)

---

**Q7: What is the Idle task in FreeRTOS and what does it do?**

> The Idle task (priority 0, lowest) runs whenever no application tasks are ready. It:
> 1. Frees memory of any tasks deleted with `vTaskDelete()`
> 2. Calls `vApplicationIdleHook()` if configured (user-defined: enter low-power mode)
> 3. Executes `__WFI()` (Wait For Interrupt) if configured — halts CPU until next interrupt
>
> **Critical rule**: If your application has a task at priority 0 that never blocks, the Idle task never runs — deleted task memory never freed, potential memory leak.

---

**Q8: Explain the difference between `vTaskDelay()` and `vTaskDelayUntil()`.**

> `vTaskDelay(100)`: Delay 100 ticks **from when the function is called**. If the task takes variable time to reach the delay call, the period is variable.
>
> `vTaskDelayUntil(&xLastWakeTime, 100)`: Delay until a tick count 100 ticks after `xLastWakeTime`, then update `xLastWakeTime`. Period is fixed regardless of task execution time.
>
> For periodic tasks (sensor sampling, control loops), **always use `vTaskDelayUntil()`** — it gives you a stable, drift-free period. `vTaskDelay()` drifts over time.

---

**Q9: What is stack overflow detection in FreeRTOS and how does it work?**

> FreeRTOS supports two levels (`configCHECK_FOR_STACK_OVERFLOW`):
>
> **Level 1**: At each context switch, check if the current stack pointer is below the stack bottom. Catches large overflows quickly.
>
> **Level 2**: At task creation, fill the top N words of the stack with a known pattern (0xA5A5A5A5). At each context switch, check if this pattern is intact. Catches gradual overflows that Level 1 misses.
>
> Both call `vApplicationStackOverflowHook(task, name)` when detected. **Warning**: By the time overflow is detected, the stack may already be corrupted — don't use FreeRTOS APIs inside the hook.

---

**Q10: What happens if `vTaskStartScheduler()` returns?**

> It only returns in two scenarios:
> 1. **Not enough heap** to create the Idle task and (if using software timers) the Timer task. This is the most common cause — increase `configTOTAL_HEAP_SIZE`.
> 2. **Implementation-specific error** (rare).
>
> The return should be treated as fatal. Add `configASSERT(0)` after `vTaskStartScheduler()` to catch it immediately in development.

---

**Q11: How do you communicate between a FreeRTOS task and an interrupt handler?**

> Use the `FromISR` variants of FreeRTOS APIs:
> ```c
> // In ISR:
> BaseType_t xHigherPriorityTaskWoken = pdFALSE;
> xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);
> xSemaphoreGiveFromISR(xSemaphore, &xHigherPriorityTaskWoken);
> // After all API calls:
> portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
> ```
> The `xHigherPriorityTaskWoken` flag tells FreeRTOS if the API unblocked a higher-priority task. `portYIELD_FROM_ISR` triggers an immediate context switch if so, keeping the system responsive.

---

**Q12: Design a producer-consumer system with FreeRTOS for a sensor that samples at 100Hz.**

> ```
> Architecture:
>
> [Sensor ISR @ 100Hz]
>   → xQueueSendFromISR(xSensorQueue, &sample, &woken)
>   → portYIELD_FROM_ISR(woken)
>
> [Processing Task - High Priority]
>   for(;;) {
>       xQueueReceive(xSensorQueue, &sample, portMAX_DELAY)
>       process(sample) // Fast processing
>       xQueueSend(xResultQueue, &result, 0)
>   }
>
> [Logging Task - Low Priority]
>   for(;;) {
>       xQueueReceive(xResultQueue, &result, portMAX_DELAY)
>       log_to_flash(result) // Can be slow
>   }
> ```
>
> Key decisions: Queue depth for `xSensorQueue` must handle worst-case burst (e.g., 10 samples = 100ms of log latency). Use `vTaskDelayUntil` if the ISR approach has jitter issues. `portMAX_DELAY` on receives ensures tasks sleep when no data available.

---

*← [[../MOC_Interview|Interview Preparation]]*
