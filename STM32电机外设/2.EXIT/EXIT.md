# 1.任务

按一下按键，触发外部中断，使LED灯电平反转

## 2.Cubemx配置

### 2.1 首先配置LED引脚输出：
<image src="LED引脚配置.png">

### 2.2 外部中断EXIT配置：

<image src="EXIT配置.png">

#### 参数

(1) GPIO mode 中断触发方式

* 可以配置为外部**中断**模式 上升沿触发或下降沿触发或上升和下降沿都触发
* 可以配置为外部**事件**模式 上升沿触发或下降沿触发或上升和下降沿都触发

(2) GPIO Pull-up/Pull-dowm

* 配置内部上下拉，这里设置为既不需要上拉也不需要下拉

(3) User Label 该GPIO引脚相关的用户标签

### 2.3 外部中断使能：
<image src="使能外部中断.png">

## 使用外部中断


中断回调函数`HAL_GPIO_EXTI_Callback`是定义在stm32g4xx_hal_gpio.c中的一个弱函数：

    __weak void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)

中断发生时，回调函数被`void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin)`调用，而`IRQHandler`又被stm32g4xx_it.c中真正的中断线函数`void EXTI15_10_IRQHandler(void)`调用起来：

```c++
void EXTI15_10_IRQHandler(void)
{
  /* USER CODE BEGIN EXTI15_10_IRQn 0 */

  /* USER CODE END EXTI15_10_IRQn 0 */
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_10);
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_11);
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);
  /* USER CODE BEGIN EXTI15_10_IRQn 1 */

  /* USER CODE END EXTI15_10_IRQn 1 */
}
```

也就是说，当中断线上发生中断时，最先进入的是`EXTI15_10_IRQHandler`函数，它并不清楚到底是发生了是哪个引脚触发了中断，因为好几个中断都连接到同一个中断向量。只能调用`HAL_GPIO_EXTI_IRQHandler`并依次传入所有可能的引脚。

```c++
void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin)
{
  /* EXTI line interrupt detected */
  if (__HAL_GPIO_EXTI_GET_IT(GPIO_Pin) != 0x00u)
  {
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_Pin);
    HAL_GPIO_EXTI_Callback(GPIO_Pin);
  }
}
```

在`HAL_GPIO_EXTI_IRQHandler`中检查到底是不是该引脚触发的中断，如果是，就调用回调函数`HAL_GPIO_EXTI_Callback`。回调函数是个弱函数，由用户重写。

我们在main.c中重写这个函数：

```c++

/* USER CODE BEGIN 4 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  /* Prevent unused argument(s) compilation warning */
  UNUSED(GPIO_Pin);
	if(Button1_Pin == GPIO_Pin)
	{
        HAL_GPIO_TogglePin(LED1_GPIO_Port,LED1_Pin);// 反转电平
	}
	if(Button2_Pin == GPIO_Pin)
	{
		HAL_GPIO_TogglePin(LED2_GPIO_Port,LED2_Pin);
	}
	if(Button3_Pin == GPIO_Pin)
	{
		HAL_GPIO_WritePin(LED1_GPIO_Port,LED1_Pin,GPIO_PIN_RESET);
		HAL_GPIO_WritePin(LED2_GPIO_Port,LED2_Pin,GPIO_PIN_RESET);
	}
  /* NOTE: This function should not be modified, when the callback is needed,
           the HAL_GPIO_EXTI_Callback could be implemented in the user file
   */
}

```


