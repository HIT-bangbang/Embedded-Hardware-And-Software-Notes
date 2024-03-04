BSP:Board Support Package 针对具体电路板的板级支持包
MSP:MCU Specific Package 针对具体MCU的初始化程序代码


1、我们要初始化和MCU无关的东西例如串口协议，其中包括波特率，奇偶校验，停止位等等，这些设置和使用什么样的MCU没有任何关系，可以使用STM32F1的MCU，也可以是F2…F4。这里只是规定了如何收发，但是并没有限定具体使用什么硬件。

2、有了抽像的串口，这个“串口”就要在MCU上进行承载，用STM32进行承载，PA9做为发送，PA10做为接收。MSP就是要初始化PA9,PA10。配置这两个引脚，例如配置引脚的输入输出模式，时钟，上下拉，以及读写数据时具体的引脚电平是什么。


所以在HAL库中，对于某个外设的驱动的.c文件里面，初始化函数分为了XXX_Init()和XXX_MSP_Init()，

XXX_Init()就是做一些与物理硬件无关的配置，例如串口中的波特率，奇偶校验，停止位，CAN通信中的滤波器，帧类型，FIFO等。

而在XXX_MSP_Init()中，具体对使用到的物理引脚进行初始化，对时钟，中断进行配置，这些都是与具体的芯片有关系的。

以g431的fdcan为例：



HAL_FDCAN_MspInit()是一个__weak函数（定义在stm32g4xx_hal_fcan.c），在fdcan.c中被重写。在stm32g4xx_hal_fcan.c中被HAL_FDCAN_Init()调用，而HAL_FDCAN_Init()又在前面说的外设驱动fdcan.c中文件中的MX_FDCAN1_Init()中被调用。

也就是说，我们只需要把驱动补充写好，然后需要在main函数中调用MX_FDCAN1_Init()就完成对协议层和MSP的所有配置。