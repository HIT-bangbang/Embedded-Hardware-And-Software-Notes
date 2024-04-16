# 任务

按一下按键，触发外部中断，使LED灯电平反转

所有芯片通用，以下以stm32f103cbt6为例

## 2.Cubemx配置

按键用于控制引脚输入电平，配置输入引脚，内部上拉，并添加用户标签为KEY：
<image src="输入引脚.png">

(1).GPIO mode ( GPIO 输入模式)

* Input mode 输入模式

(2).GPIO Pull-up/Pull-dowm (上拉下拉电阻)

* No pull-up and no pull-down无上拉或下拉
* pull-up 内部上拉电阻
* Pull-dowm 内部下拉电阻

输出到LED引脚配置如下，推挽输出，并添加用户标签为LED：
<image src="输出引脚.png">

### 2.2 GPIO输出参数解释

(1).GPIO output level (引脚初始电平设置 )

* High 输出初始化为高电平
* Low 输出初始化为低电平

(2).GPIO mode ( GPIO 输出模式)
* Output Push pull 推挽输出，可以输出较大电流，并允许较大电流流入，可以为IO设备供电
* Output Open Drain 开漏输出模式 只有低电平可以作为负载工作的负极，但是高电平时为高阻态，只可以作为一个信号输出

(3).GPIO Pull-up/Pull-dowm (上拉下拉电阻)

* No pull-up and no pull-down无上拉或下拉
* pull-up 内部上拉
* Pull-dowm 内部下拉

(4).Maxinum output speed（引脚速度设置）

* Low 低速
* Medium 中速
* High 高速
* Very High 高速

(5).User Label(用户标签)

* 在代码中为该GPIO设置宏定义别名LED0


## 3.主要函数


读取引脚电平：

    HAL_GPIO_ReadPin(KEY_GPIO_Port,KEY_Pin)

返回值为`GPIO_PinState`类型，可能的值为`GPIO_PIN_SET` 或 `GPIO_PIN_RESET`

设置引脚电平：

    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET)
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET)

反转电平：

	HAL_GPIO_TogglePin(LED_GPIO_Port,LED_Pin);

由于我们设置了用户标签，Cubemx生成代码时会自动帮我们进行宏定义，给GPIO和PIN起别名。`KEY_GPIO_Port` `LED_GPIO_Port` `KEY_Pin` `LED_Pin`


`GPIO_PIN_SET` `GPIO_PIN_RESET` 是HAL库中已经定义好的宏，SET表示1，RESET表示0

例程：

```c++
  while (1)
  {
		if(HAL_GPIO_ReadPin(KEY_GPIO_Port,KEY_Pin) == GPIO_PIN_RESET)
		{
			while(HAL_GPIO_ReadPin(KEY_GPIO_Port,KEY_Pin) == GPIO_PIN_RESET);
			HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);
			
			HAL_Delay(500);
			
			HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);

		}
  }

```