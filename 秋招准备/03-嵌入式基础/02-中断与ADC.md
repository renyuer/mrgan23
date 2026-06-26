## 中断（Interrupt）— 嵌入式面试灵魂考题

### 中断处理流程
```
外设事件 → NVIC → 查中断向量表 → 跳转 ISR → 执行 → 返回
         → 保护现场（自动压栈 R0-R3,R12,LR,PC,xPSR）
         → 恢复现场（自动出栈）
```

### NVIC 优先级
```c
// STM32：4位优先级位
// 可以配置为：
// - 0 位抢占 + 4 位子优先级（16级子优先级，无抢占嵌套）
// - 4 位抢占 + 0 位子优先级（16级抢占嵌套）
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);  // 4位全给抢占

HAL_NVIC_SetPriority(EXTI0_IRQn, 0, 0);   // 抢占优先级 0（最高）
HAL_NVIC_SetPriority(TIM2_IRQn, 1, 0);    // 抢占优先级 1
```

### ISR 编写铁律（面试必考）

**铁律1：快进快出**
```c
// ❌ 错误：ISR 里做重活
void EXTI_IRQHandler(void) {
    HAL_Delay(100);           // 阻塞太久
    printf("interrupt!\n");   // printf 可能阻塞
    process_big_data();       // 耗时
}

// ✅ 正确：ISR 只置标志，主循环处理
volatile uint8_t g_data_ready = 0;

void EXTI_IRQHandler(void) {
    g_data_ready = 1;  // 只干这一件事
}

void main_loop(void) {
    if (g_data_ready) {
        g_data_ready = 0;
        process_big_data();  // 主循环里慢处理
    }
}
```

**铁律2：ISR 中不能用会阻塞的 API**
- `HAL_Delay()` 依赖 SysTick 中断，如果当前中断优先级高于 SysTick，永远等不到
- `printf()` 可能等 UART 发送完成，阻塞
- 应使用 `osSignalSet()`（FreeRTOS 的 FromISR 版本）或队列发送 `xQueueSendFromISR()`

**铁律3：共享变量加 volatile**
```c
volatile uint8_t flag;  // ISR 写，主循环读
```

**铁律4：注意中断嵌套**
- 高优先级中断可以抢占低优先级
- 同一优先级的中断不可相互抢占
- FreeRTOS 中，中断优先级不能高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`

---

## ADC — 模拟到数字

### 关键参数
| 参数 | 含义 | STM32F103 |
|------|------|-----------|
| 分辨率 | ADC 转换精度位数 | 12 位（0~4095） |
| 参考电压 | 满量程对应的电压 | 通常 3.3V |
| 采样率 | 每秒转换次数 | 最高 1Msps |
| 通道数 | 可接几个模拟输入 | 最多 16+2 |

### 电压计算
```
数字值 = (输入电压 / 参考电压) × (2^分辨率 - 1)

例：3.3V 参考，12 位 ADC，读数 2048
电压 = 2048 / 4095 × 3.3V ≈ 1.65V
```

### 面试必问：DMA + ADC 连续采集
你飞控遥控端摇杆采集就是这个：

```c
// ADC + DMA 连续模式
HAL_ADC_Start_DMA(&hadc1, (uint32_t *)adc_buf, NUM_CHANNELS);

// adc_buf[] 自动被 DMA 循环填充
// 主循环直接读 buf，不阻塞，零 CPU 开销
uint16_t throttle = adc_buf[0];
uint16_t yaw      = adc_buf[1];
uint16_t pitch    = adc_buf[2];
uint16_t roll     = adc_buf[3];
```

### ADC 的采样时间
- STM32 可配置不同的采样周期（1.5/7.5/13.5/28.5/... 个 ADC 时钟周期）
- 采样时间越长，对高阻抗信号源越友好
- 采样时间 + 12.5 个 ADC 周期 = 一次转换总时间

### 面试题：ADC 读数跳动怎么办？
1. **硬件**：电源滤波、信号线上加 RC 低通
2. **软件**：滑动平均滤波、中值滤波、一阶低通（你在飞控项目里做了）
3. **采样策略**：多次采样取平均、丢弃异常值

---

## DMA（Direct Memory Access）

### 核心思想
不需要 CPU 参与的数据搬运，外设直接和内存交互

### STM32 DMA 配置要素
```c
// 用 DMA 把 ADC 数据搬到数组
hdma_adc.Instance = DMA1_Channel1;
hdma_adc.Init.Direction = DMA_PERIPH_TO_MEMORY;   // 外设 → 内存
hdma_adc.Init.PeriphInc = DMA_PINC_DISABLE;       // 外设地址不变
hdma_adc.Init.MemInc = DMA_MINC_ENABLE;           // 内存地址递增
hdma_adc.Init.Mode = DMA_CIRCULAR;                // 循环模式
```

### 面试常问：DMA 能减轻什么？
- 减轻 CPU 负担：CPU 不用一次又一次响应每个字节的传输中断
- 典型场景：ADC 连续扫描、UART 大数据收发、SPI 屏刷图

---

## 自我检测

1. ISR 中不能做什么？为什么？
2. ISR 和主循环共享变量，为什么加 volatile 还不够？
3. 12 位 ADC，参考电压 3.3V，读数为 2048，对应电压是多少？
4. ADC + DMA 连续采集相比轮询有什么优势？
5. FreeRTOS 中，ISR 能直接调用 `xQueueSend()` 吗？应该用什么？
