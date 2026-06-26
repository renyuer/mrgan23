## GPIO — 面试问最多的外设

### 四种输出模式
| 模式 | 特点 | 典型用途 |
|------|------|----------|
| 推挽输出 | 高低电平都能驱动，驱动力强 | LED、继电器 |
| 开漏输出 | 只能拉低，不能拉高，需外部上拉 | I2C、电平转换 |
| 复用推挽 | 外设控制输出（SPI/UART等） | 通信引脚 |
| 复用开漏 | 外设控制 + 开漏特性 | I2C 的 SDA/SCL |

### 三种输入模式
| 模式 | 特点 |
|------|------|
| 浮空输入 | 无上下拉，悬空时电平不确定 |
| 上拉输入 | 内部上拉电阻，悬空默认高电平 |
| 下拉输入 | 内部下拉电阻，悬空默认低电平 |

### 面试必问
**Q: I2C 为什么用开漏输出？**
A: ① 实现线与逻辑，多个设备可以共享总线（任一设备拉低，总线就低）② 支持双向通信（主从都能控制 SDA）③ 开漏+上拉可以实现电平转换

**Q: GPIO 输出速度（2MHz/10MHz/50MHz）是什么意思？**
A: 不是输出信号的频率上限，而是 GPIO 驱动电路的翻转速度/斜率控制——速度越快，IO 翻转斜率越陡，EMI 越大但能满足高频通信；低速模式 EMI 小但波形可能失真

### STM32 HAL 快速回顾
```c
// 初始化
GPIO_InitTypeDef g = {0};
g.Pin = GPIO_PIN_13;
g.Mode = GPIO_MODE_OUTPUT_PP;  // 推挽输出
g.Pull = GPIO_NOPULL;
g.Speed = GPIO_SPEED_FREQ_LOW;
HAL_GPIO_Init(GPIOC, &g);

// 操作
HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
GPIO_PinState s = HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_0);
```

---

## 定时器 — 嵌入式时间基准

### 三种定时器
| 类型 | 位数 | 功能 |
|------|------|------|
| 基本定时器 | 16位 | 仅时基（定时） |
| 通用定时器 | 16/32位 | PWM输出、输入捕获、编码器 |
| 高级定时器 | 16位 | 带死区互补PWM、刹车（电机控制） |

### PWM 输出 — 面试高频

**核心参数**
- **频率** = 定时器时钟 / (PSC+1) / (ARR+1)
- **占空比** = CCR / (ARR+1) × 100%

```c
// 以 STM32F103，72MHz 为例
// 目标：1kHz PWM，占空比可调

htim.Instance = TIM2;
htim.Init.Prescaler = 71;      // 72MHz / (71+1) = 1MHz
htim.Init.Period = 999;        // 1MHz / (999+1) = 1kHz
htim.Init.CounterMode = TIM_COUNTERMODE_UP;

// 某通道输出 30% 占空比
__HAL_TIM_SET_COMPARE(&htim, TIM_CHANNEL_1, 300);
// CCR / (ARR+1) = 300 / 1000 = 30%
```

**Q: 怎么改变 PWM 占空比？**
A: 修改 CCR 寄存器（Capture/Compare Register）

**Q: 怎么改变 PWM 频率？**
A: 修改 ARR（自动重装载值）——但同时会改变占空比，因为占空比 = CCR/ARR，需要按比例同时调整 CCR

### 输入捕获（Input Capture）
你闹钟项目里声控唤醒测脉宽就是这个：
```c
// 测量信号脉宽
// 上升沿 → 记录时间 T1 → 下降沿 → 记录时间 T2
// 脉宽 = (T2 - T1) × 定时器周期
```

### 面试追问：PWM 的分辨率由什么决定？
A: 由 ARR 决定。ARR 越大，分辨率越高。16 位定时器 ARR 最大 65535，提供 0~65535 级的占空比精度。

---

## 自我检测

1. 开漏输出和推挽输出的本质区别？各举一个使用场景
2. STM32 如何配置一个 50Hz 的 PWM，定时器时钟 72MHz？
3. PWM 频率和占空比分别由哪个寄存器控制？
4. 输入捕获测量脉宽的原理（文字 + 时序图）
5. I2C 为什么必须用开漏输出？
