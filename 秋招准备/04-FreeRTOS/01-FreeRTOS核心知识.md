## 你的飞控用了 FreeRTOS — 面试必深挖

> 简历原文："移植FreeRTOS划分飞控、姿态解算、通信等5个任务，vTaskDelayUntil保证控制周期确定性，队列传数据，互斥锁保护共享状态，任务栈余量运行时验证"
> 每一句话都是面试官追问的入口。下面逐一拆解。

---

## 任务调度

### 任务状态转换（必背）
```
         创建
          │
          ▼
     ┌─ Ready ───────────┐
     │                    │
     │ 调度器选中          │ 时间片到/被抢占
     ▼                    │
  Running ───→ Blocked   │
     │        (等信号量/延时/队列)│
     │          │               │
     │          ▼               │
     │        Ready ◄───────────┘
     │         事件到来
     ▼
  Suspended (需手动恢复)
```

### 抢占式调度
- 高优先级任务就绪 → 立即抢占低优先级任务
- 你的飞控：姿态解算任务优先级 > 通信任务 > OLED 显示

### vTaskDelay vs vTaskDelayUntil（关键！你简历里写了）

```c
// vTaskDelay：相对延迟——会累积误差
void TaskA(void *pv) {
    while (1) {
        do_work();              // 耗时不确定
        vTaskDelay(pdMS_TO_TICKS(10));  // 从此刻起再等10ms
        // 实际周期 = 10ms + do_work()耗时 → 周期漂移
    }
}

// vTaskDelayUntil：绝对延迟——周期精确
void TaskB(void *pv) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    while (1) {
        do_work();              // 耗时不确定
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(10));
        // 实际周期 = 精确10ms（前提是 do_work 在10ms内完成）
    }
}
```

**面试话术**：飞控任务必须每个控制周期精确同步，如果周期漂移，PID 控制就会抖动。所以用 `vTaskDelayUntil` 而不是普通 delay。

---

## IPC（任务间通信）

### 队列 — 你飞控用"队列传数据"
```c
QueueHandle_t xQueue = xQueueCreate(10, sizeof(sensor_data_t));

// 任务A：发送
sensor_data_t data = { .accel_x = 1000, .gyro_z = 50 };
xQueueSend(xQueue, &data, portMAX_DELAY);

// 任务B：接收
sensor_data_t received;
xQueueReceive(xQueue, &received, portMAX_DELAY);
```

**面试必问**：队列传的是拷贝还是引用？
- **拷贝**（默认）：数据从发送方内存拷贝到队列缓冲区。安全，但大结构体开销大
- **引用**（传指针）：只拷贝指针本身。效率高，但要注意：发送方不能提前修改该内存！需配合信号量或确保数据用完才改

### 信号量 — 同步和资源计数

| 类型 | 用途 | 典型场景 |
|------|------|----------|
| 二值信号量 | 任务同步（事件通知） | ISR 通知任务"数据就绪" |
| 计数信号量 | 管理多个资源 | 管理 N 个 UART 缓冲区 |
| 互斥锁 | 保护共享资源 | 保护全局传感器数据 |

```c
// 二值信号量：ISR → 任务同步
SemaphoreHandle_t xSem = xSemaphoreCreateBinary();

void ISR(void) {
    xSemaphoreGiveFromISR(xSem, NULL);  // ISR 里发信号
}

void Task(void *pv) {
    if (xSemaphoreTake(xSem, portMAX_DELAY)) {
        process_data();  // 收到信号才处理
    }
}
```

### 互斥锁 vs 信号量（面试高频对比）

| 对比项 | 互斥锁 | 二值信号量 |
|--------|--------|-----------|
| 所有权 | 谁 lock 谁 unlock | 无所有权（ISR 可以 give） |
| 优先级继承 | 有（防止优先级反转） | 无 |
| 递归 | 支持（递归互斥锁） | 不支持 |
| 用途 | 保护共享资源 | 任务同步/事件通知 |

**你的飞控项目**："互斥锁保护共享状态"
```c
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();

void Task_Sensor(void *pv) {
    xSemaphoreTake(xMutex, portMAX_DELAY);
    update_sensor_data();  // 临界区
    xSemaphoreGive(xMutex);
}
```

### 优先级反转及解决（必考）

**场景**：
- 高优先级任务 H 等互斥锁
- 锁被低优先级任务 L 持有
- 中优先级任务 M 一直抢占 L → L 永远跑不完 → H 永远等不到锁

**解决**：**优先级继承**
- FreeRTOS 互斥锁自动启用：当 H 等锁时，L 的优先级被临时提升到 H 的级别
- L 快速跑完释放锁 → H 拿到锁 → L 优先级恢复

---

## 任务栈与溢出检测

### 你的简历："任务栈余量运行时验证"

**怎么做**：
```c
// 方法1：uxTaskGetStackHighWaterMark()
UBaseType_t uxHighWaterMark = uxTaskGetStackHighWaterMark(NULL);
printf("Min free stack: %lu words\n", uxHighWaterMark);
// 返回值是任务运行以来最小的剩余栈空间（单位：字）

// 方法2：栈填充检测
// 创建任务时用 0xA5 填充栈空间
// 定期扫描被覆盖的区域来计算栈最大使用量
```

**面试话术**："每个 FreeRTOS 任务创建时分配了栈空间，我定期调用 `uxTaskGetStackHighWaterMark()` 来检查运行时还有多少栈余量，确保不会溢出导致 HardFault。同时在开发阶段故意压测，确认每个任务的栈大小设置合理。"

### 栈溢出检测钩子
```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    // 栈溢出时自动调用
    // 开发阶段在这里打日志/断点，发布阶段进入安全状态
    while (1);
}
```
需在 FreeRTOSConfig.h 中开启 `configCHECK_FOR_STACK_OVERFLOW`

---

## 内存管理（heap_1 ~ heap_5）

| 方案 | 特点 | 适用场景 |
|------|------|----------|
| heap_1 | 只分配不释放，最简单 | 只创建不删除任务 |
| heap_2 | 可释放，但不会合并碎片 | 创建/删除大小固定的对象 |
| heap_3 | 包装 stdlib malloc/free | 需要标准库实现 |
| heap_4 | 会合并相邻空闲块（推荐） | 大多数项目首选 |
| heap_5 | heap_4 + 跨多块内存区域 | 片外 RAM + 片内 RAM 混合 |

**你的飞控**：代码 < 12KB RAM，32KB Flash，大概率 heap_4

---

## 临界区保护

### 两种方式
```c
// 方式1：关中断（最简单，但影响中断响应）
taskENTER_CRITICAL();
// 临界区代码（不可被中断打断）
taskEXIT_CRITICAL();

// 方式2：挂起调度器（不关中断，但阻止任务切换）
vTaskSuspendAll();
// 临界区代码（不可被其他任务打断，但 ISR 可响应）
xTaskResumeAll();
```

**对比**：关中断更安全（连 ISR 都不能抢），但增加中断延迟。挂起调度器让 ISR 能及时响应，但 ISR 不能调 FreeRTOS API。

---

## 自我检测

1. vTaskDelay 和 vTaskDelayUntil 的区别？飞控为什么用后者？
2. 队列默认传的是拷贝还是引用？各有什么优缺点？
3. 互斥锁和信号量的本质区别？什么时候用哪个？
4. 优先级反转怎么产生？FreeRTOS 怎么解决？
5. 栈溢出检测有哪几种方法？你的飞控怎么做的？
6. heap_4 和 heap_1 的区别？
7. 临界区保护两种方式的区别和适用场景？