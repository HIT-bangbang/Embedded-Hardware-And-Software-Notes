# 一、阻塞(轮询)收发

## 1.1 Cubemx配置

<image src="cubemx配置.png">

(1) Mode

* 设置为异步模式 Asynchronous

(2) 硬件流控制 Hardware Flow Control

* 设置为Disable

(3) Baud Rate 波特率

(4) World Lenth 

* 一般是 8 Bit

(5) parity  校验位

* 设置为none

(6) stop bit  停止位

一般设置为 1

## 1.2 keil使用

### (1)阻塞式发送指定个数字节数据

    HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart, const uint8_t *pData, uint16_t Size, uint32_t Timeout);

这里发送数据的指针需要是`uint8_t *`类型，否则会报warning，但是无关紧要，可以忽略。也可以将其他类型数据的指针进行强制类型转换。

最后一个参数是超时时间，最大值为`HAL_MAX_DELAY`表示一直等待

### (2)阻塞式接收指定个数字节数据

    HAL_StatusTypeDef HAL_UART_Receive(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size, uint32_t Timeout);


### (3)重定向printf

* 1. 首先勾选`MicroLIB`选项

<image src="MicroLIB.png">

* 2. 重写fputc函数(main.c中，其他文件也可，但是注意包含关系)
  
```c++
int fputc(int ch, FILE *f){
	uint8_t temp[1] = {ch};
	HAL_UART_Transmit(&huart1, temp, 1, 2);//huart1需要根据你的配置修改
	return ch;
}
```

* 3. 使用printf函数

### (4)可变宏快速Debug调试格式化

使用可变参数宏

```c++
#define USER_MAIN_DEBUG
#ifdef USER_MAIN_DEBUG
#define user_main_printf(format, ...) printf( format "\r\n", ##__VA_ARGS__)
#define user_main_info(format, ...) printf("[\tmain]info:" format "\r\n", ##__VA_ARGS__)
#define user_main_debug(format, ...) printf("[\tmain]debug:" format "\r\n", ##__VA_ARGS__)
#define user_main_error(format, ...) printf("[\tmain]error:" format "\r\n",##__VA_ARGS__)
#else
#define user_main_printf(format, ...)
#define user_main_info(format, ...)
#define user_main_debug(format, ...)
#define user_main_error(format, ...)
#endif
```
然后就可以使用`user_main_printf` `user_main_info` `user_main_debug` 等已经写好的格式进行Debug输出了

# 二、中断收发

打开USART的中断功能：

<image src="uart中断.png">
在这里也行
<image src="uart中断2.png">


### 中断发送函数：

    HAL_StatusTypeDef HAL_UART_Transmit_IT(UART_HandleTypeDef *huart, const uint8_t *pData, uint16_t Size);

中断发送函数可以直接使用，和轮询发送一样用

### 中断接收函数：

    HAL_StatusTypeDef HAL_UART_Receive_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size);

该函数其实是配置接收中断，只要在`While(1)`循环的前面执行一次即可。

接收到数据的处理放在中断回调函数中。类似外部中断，重定义弱函数：
    __weak void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)

并且在中断回调函数中对数据进行处理。

```c++
//开启中断接收
HAL_UART_Receive_IT(...)

while(1)
{
    .....
}
HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    //发回接收到的数据
    HAL_UART_Transmit_IT(...)
    //处理接收到的数据
    ...

    //重新开启中断接收
    HAL_UART_Receive_IT(...)
}

```


# 三、DMA收发


点击ADD，创建DMA通道，一个用于发送，一个用于接收。参数保持默认即可


发送的数据搬运方向是：`Memory To Peripheral`
接收的数据搬运方向是：`Memory To Peripheral`

优先级都是 LOW

模式选择为 `normal`

地址自增选项中：外设中数据地址不自增，内存中要自增。

数据宽度都是字节，因为每次搬运都是一个字节一个字节

## DMA发送

直接使用DMA发送函数即可：

    HAL_StatusTypeDef HAL_UART_Transmit_DMA(UART_HandleTypeDef *huart, const uint8_t *pData, uint16_t Size);


## DMA接收

DMA接收函数用法与中断接收函数相同，改一下名字即可。
```c++
//开启中断接收
HAL_UART_Receive_DMA(...)

while(1)
{
    .....
}
HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    //发回接收到的数据
    HAL_UART_Transmit_DMA(...)
    //处理接收到的数据
    ...

    //重新开启中断接收
    HAL_UART_Receive_DMA(...)
}
```
可以看到，DMA接收也在中断回调函数中进行操作，但是此时的中断就是DMA传输完成中断了，而不是中断接收中使用的发送数据寄存器空中断/接收数据寄存器非空中断了。


## DMA接收不定长数据




