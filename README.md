# 电控组第二次培训作业

- 姓名：**弓顺开**
- 学号：***20253006793***

## 流水灯实现思路

用 CubeMX 把 PA0、PA1、PA2 配置成输出，User Label 分别命名为 LED0、LED1、LED2，
高电平点亮。代码里统一用生成的 `LED0_Pin` / `LED0_GPIO_Port` 这些宏操作引脚。

`LED_OnStep(step)` 负责点亮第 step 颗灯，内部先调用 `LED_AllOff()` 把三颗全灭，
再点亮当前那颗，step 在 0→1→2 之间循环，每次切换间隔 500ms。

## 非阻塞延时

没有用 `HAL_Delay()`，因为它内部是 `while` 死等，这 500ms 里 CPU 一直空转，
没法干别的事。

两个版本都做了非阻塞，区别在于**时间由谁来数**。

### A 版：`HAL_GetTick()` 时间戳（version_a_tick）

主循环里不停比较当前时间和上次切换的时刻，够了才切灯，没到就直接进入下一圈。

```c
static uint32_t last_ms = 0u;
static uint8_t  step    = 0u;

while (1)
{
  if ((HAL_GetTick() - last_ms) >= 500u)
  {
    last_ms = HAL_GetTick();
    step = (uint8_t)((step + 1u) % 3u);
    LED_OnStep(step);
  }
}
```

时间差用无符号相减，HAL_GetTick() 溢出时结果也不会错。

### B 版：TIM2 定时器中断（version_b_tick）

让定时器硬件自己数数，数到 500ms 主动进中断切灯，主循环什么都不用做。

PSC 设 7199、ARR 设 4999，72MHz 下正好 500ms 进一次中断：

```c
static uint8_t s_step = 0u;

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  if (htim->Instance == TIM2)
  {
    s_step = (uint8_t)((s_step + 1u) % 3u);
    LED_OnStep(s_step);
  }
}
```

```c
LED_OnStep(0u);
HAL_TIM_Base_Start_IT(&htim2);
```

## 工程说明

仓库里两个版本都是完整可编译的 CubeMX 工程，直接打开对应目录即可：

- `version_a_tick/` —— A 版
- `version_b_tick/` —— B 版
- `video/` —— 两个版本的验收视频

- [A 版验收视频](https://github.com/zcpym/homework2/raw/main/video/version_a_demo.mp4)
- [B 版验收视频](https://github.com/zcpym/homework2/raw/main/video/version_b_demo.mp4)
