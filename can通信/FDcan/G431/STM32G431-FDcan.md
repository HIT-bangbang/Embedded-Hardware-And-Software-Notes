# STM32G431为例 Classic CAN通信

STM32G4系列和H7的can均为FD-can（支持FDcan，兼容经典can）

stm32f1和F4的叫做bxCAN（Basic Extended CAN）

在硬件架构上和软件配置上都不一样

# Message RAM

FDcan的数据段长度最高可以支持到64bytes，像f1和f4那种bxCAN的接收发送邮箱、滤波器存储空间设计就很局限了。所以在G4和H7的FDcan控制器中，单独开辟了一块RAM用于所有can message的存储。
以下是G431的FDcan Message RAM 存储空间的分配：
<image src="G431-message-ram.png">

G431只有一个FDcan，独享这一片RAM存储区域，该RAM区域中已经为各个部分分配好了固定的空间大小。

一个element也就是一个元素，可以是一个滤波器或者是FIFO中存的一个报文等等。一个element不一定占多大的空间，这个根据element的类型来看。

图中的各个区域也不是都需要配置，因为有些区域可以不用。例如我们不使用FIFO1的话，就不用管它。

对于stm32，32位MCU一个words是32位（32bit）

图中也标明了各部分的地址，对于有两个can的G4系列芯片，手册44.3.3 Message RAM也说明了如何计算地址。

通过该图可以看出：

* 最多可以配置0~28个11位标准帧ID过滤器
* 最多可以配置0~8个29位扩展帧ID过滤器
* Rx FIFO 0 的深度为3，也就是最多存3条报文，这个三条报文总共的长度不能超过54个word
* Rx FIFO 1 的深度为3，也就是最多存3条报文，这个三条报文总共的长度不能超过54个word
* Tx event FIFO 的深度为3，也即是最多存3个发送事件， 总共的占用空间不能超过6个word
* Tx buffers 深度为3，也就是最多存3条报文，这个三条报文总共的长度不能超过54个word

一个报文或者event或者滤波器到底占多少个world，会不会溢出不需要我们操心，因为空间已经分配好了是固定的，而且是足够的，所以在后面的CubeMx和Keil配置中，也都不需要提前配置好报文的长度等等（不像H7的FDcan一样，每个区域的占用空间大小是浮动可调的）

其实这块RAM的设计和bxCAN很相似，只是彻底放弃了所谓的Mailbox邮箱的说法，因为本质上邮箱就是一块存储空间，就是一个FIFO。

## 1、CubeMX配置

### 1.1 时钟

can的时钟有三个来源

<image src="clock.png">

### 1.2 协议参数配置

<image src="param.png">

* Clock Divider：时钟分频
* Frame Format：配置经典can（classic）、带有BitRate切换的FD-can、不带BitRate切换的FD-can
* Mode: 工作模式，见参考手册
* Auto Retransmission：是否启用自动重发
* Transmit Pause：使能传输暂停功能（应该是为了避免一些优先级过高的消息长时间抢占总线）
* Protocol Exception： 协议异常使能（应该只用于FDcan，以后再研究）
* Nominal Sync Jump Width: Nominal重新同步跳跃宽度(SJW)

* Data Prescaler: 数据段时钟分频值
* Data Sync Jump Width：Data段重新同步跳跃宽度(SJW)
* Data Time Seg1: 数据段的$t_{BS1}$
* Data Time Seg2: 数据段的$t_{BS2}$

* Std Filters Nbr:标准帧ID过滤器的数量
* Ext Filters Nbr:扩展帧ID过滤器的数量
* Tx Fifo Queue Mode: 配置发送的顺序，FIFO模式按照消息写入FIFO的先后顺序配置帧的发送优先级，queue模式按照帧的ID设置消息的发送优先级，

* Nominal Prescaler: Nominal时钟分频值
* Nominal Time Seg1: Nominal的$t_{BS1}$
* Nominal Time Seg2: Nominal的$t_{BS2}$

Nominal波特率受到 ***can时钟源频率、Clock Divider 、Nominal Prescaler、Nominal Time Seg1、Nominal Time Seg2*** 的影响。

$$ Nominal Baud Rate = Clock frequence \div Clock Divider \div Nominal Prescaler \div (Nominal Time Seg1 + Nominal Time Seg2 +1) $$

**注意** SJW不参与波特率的计算，很多博文全都写错了，实际上你在Cubemx中调SJW的值会发现最终的波特率根本不变。最后那个$+1$加的其实是$t_{SyncSeg}$，在stm32中 $t_{SyncSeg} = 1 t_q$。

Frame Format配置为经典Classic CAN或不启用BitRate switching时，***Data Prescaler，Data Sync Jump Width，Data Time Seg1，Data Time Seg2*** 无效，保持默认即可。

应为FDcan可以有两个波特率，一个是名义波特率（Nominal Baud Rate），一个是data段的可变波特率。如果Frame Format启用了BitRate Switching，那么这几个参数是用来配置可变的data段的波特率的。

一定要配置好足够的Filter数量，如果写为0，则后面所有过滤器的配置都是无效的。stm32G431只有一个FD-can 可以被配置为经典can和FDcan，该can控制器有Up to 28 filters can be defined for 11-bit IDs, up to 8 filters for 29-bit IDs.

    Std Filters Nbr:标准帧ID过滤器的数量0~28
    Ext Filters Nbr:扩展帧ID过滤器的数量0~8

### 1.3 中断配置

CAN接收有两个FIFO，也有两个中断线fdcan_intr0_it和fdcan_intr1_it。FIFO0中断绑定在中断线fdcan_intr0_it上。所以中断接收也需要使能所需的中断。当然后面在keil中还需要自己详细配置中断的类型。

<image src="CAN-NVIC.png">

## 2、Keil配置

### 2.1 fdcan.c

`HAL_FDCAN_MspInit()`是一个__weak函数（定义在stm32g4xx_hal_fcan.c），在fdcan.c中被重写。在stm32g4xx_hal_fcan.c中被`HAL_FDCAN_Init()`调用，而`HAL_FDCAN_Init()`又在前面说的外设驱动fdcan.c中文件中的`MX_FDCAN1_Init()`中被调用。

所以我们的关于滤波器配置的代码也属于协议层的配置，也放在`MX_FDCAN1_Init()`函数中，这样在main.c中调用一次`MX_FDCAN1_Init()`就完成了协议层和MSP的初始化。

首先配置一下接收滤波器，在`MX_FDCAN1_Init()`用户代码区添加

```c++
  /* USER CODE BEGIN FDCAN1_Init 2 */
	
	FDCAN_FilterTypeDef FDCAN1_RXFilter;				//初始化滤波器属性结构体

	FDCAN1_RXFilter.IdType = FDCAN_STANDARD_ID;         //使用标准帧ID
    FDCAN1_RXFilter.FilterIndex = 0;                    //滤波器索引
    FDCAN1_RXFilter.FilterConfig = FDCAN_FILTER_TO_RXFIFO0;    //过滤器0关联到FIFO0
    FDCAN1_RXFilter.FilterType = FDCAN_FILTER_RANGE;    //滤波器类型，范围模式 
    FDCAN1_RXFilter.FilterID1 = 0x000;                         //过滤器边界
    FDCAN1_RXFilter.FilterID2 = 0x7FF;;                        //过滤器边界
    if(HAL_FDCAN_ConfigFilter(&hfdcan1,&FDCAN1_RXFilter) != HAL_OK)   //滤波器初始化
		{
			Error_Handler();
		}
    
    HAL_FDCAN_Start(&hfdcan1);//开启FDCAN
    
    // 使能中断回调函数FDCAN_IT_RX_FIFO0_NEW_MESSAGE
    HAL_FDCAN_ActivateNotification(&hfdcan1,FDCAN_IT_RX_FIFO0_NEW_MESSAGE,0);

  /* USER CODE END FDCAN1_Init 2 */
```

### 2.2 FDCAN_FilterTypeDef FDCAN1_RXFilter结构体参数 

下面详细讲解各个参数

* IdType: 可选FDCAN_STANDARD_ID（标准帧ID）FDCAN_EXTENDED_ID（扩展帧ID）
* FilterIndex: 滤波器索引，就是说现在是配置的哪一个滤波器，注意不要超了前面Cubemx里面配置的Std Filters Nbr或者Ext Filters Nbr。
* FilterConfig: 将当前配置的过滤器关联到哪一个FIFO，G431的CAN接收一共有两个FIFO，可选0和1

* FilterType, FilterID1, FilterID2: 配置过滤器的类型和ID1、ID2。对于标准帧，前面配置了IdType=FDCAN_STANDARD_ID, 则ID的取值范围为 0到0x7FF（正好是11位最小最大范围），同理对于扩展帧（IdType=FDCAN_EXTENDED_ID）范围是0到0x1FFFFFFF

G431的CAN接收过滤器一共有四个模式，分别为：

* FDCAN_FILTER_RANGE 一个范围过滤器，只有ID在FilterID1和FilterID2范围内的才可以通过
* FDCAN_FILTER_DUAL 两个ID过滤器，只有ID和FilterID1和FilterID2其中任意一个一样的才可以通过
* FDCAN_FILTER_MASK 一个掩码模式的过滤器，此时FilterID1配置为过滤器，FilterID2配置为掩码，在掩码为1的位置上， 报文的ID和FilterID1对应位匹配就可通过。
* FDCAN_FILTER_RANGE_NO_EIDM 也是范围过滤模式，但是不使用EIDM。这个参数只针对扩展帧。

对于扩展帧，还有一个EIDM（扩展ID掩码，对应XIDAM寄存器），通过配置EFT寄存器使能该功能，并在XIDAM寄存器设置掩码，可以在范围过滤的基础上再添加掩码，相当于FDCAN_FILTER_RANGE和FDCAN_FILTER_MASK的叠加使用。该函数为`HAL_StatusTypeDef HAL_FDCAN_ConfigExtendedIdMask(FDCAN_HandleTypeDef * hfdcan, uint32_t Mask)`。**见参考手册的44.4.20和44.3.3 Range filter**

<image src="Rangefilter.png">

例如，对于扩展帧IdType =FDCAN_EXTENDED_ID，FilterType设置为FDCAN_FILTER_RANGE时，设置FilterID1和FilterID2为0x11111111和0x12222222，同时使用`HAL_FDCAN_ConfigExtendedIdMask()`设置扩展帧ID掩码为0x0000000F（即0 0000 0000 0000 0000 0000 0000 1111）。则报文ID首先与扩展帧ID掩码进行与运算，然后再进如范围过滤器

### 2.3 使能中断回调函数

    // 使能中断线，并且配置具体触发中断事件类型FDCAN_IT_RX_FIFO0_NEW_MESSAGE
    HAL_FDCAN_ActivateNotification(&hfdcan1,FDCAN_IT_RX_FIFO0_NEW_MESSAGE,0);

其中要说的是 第二个参数，该参数用于配置中断的类型。可选参数如下：

    FDCAN_IT_RX_FIFO0_MESSAGE_LOST
    FDCAN_IT_RX_FIFO0_FULL
    FDCAN_IT_RX_FIFO0_NEW_MESSAGE

前面在Cubemx中已经使能了FIFO0的中断，但是还需要配置具体的中断类型（也就是can的中断线上面发生中断事件后，判断具体是发生了什么中断事件，因为不同类型的中断都走这一根中断线）

调用`HAL_FDCAN_ActivateNotification()`后，函数内部调用了：

    /* Enable the selected interrupts */
    __HAL_FDCAN_ENABLE_IT(hfdcan, ActiveITs);

也就是修改了`hfdcan->Instance->IE`

之后，具体的中断发生的过程是这样的，

FIFO0的接收到新的消息时，触发CAN中断0(fdcan_intr0_it)，并进入stm32g4xx_it.c中的中断函数`FDCAN1_IT0_IRQHandler();`，该函数中调用了`HAL_FDCAN_IRQHandler(&hfdcan1);`，这个函数里面进行中断事件的判断，就是判断这个CAN interrupt 0上面发生的这个中断事件到底是不是hfdcan1的中断，如果是的话是什么中断（用一些if else），发现是hfdcan里面的FIFO0中断被人配置过了，那就直接唤起回调函数：`HAL_FDCAN_RxFifo0Callback(hfdcan, RxFifo0ITs);`并将到底是哪一个hfdcan和中断的类型RxFifo0ITs作为参数传递给该回调函数。

```c++
    /* Rx FIFO 0 interrupts management ******************************************/
    if (RxFifo0ITs != 0U)
    {
        /* Clear the Rx FIFO 0 flags */
        __HAL_FDCAN_CLEAR_FLAG(hfdcan, RxFifo0ITs);

    #if USE_HAL_FDCAN_REGISTER_CALLBACKS == 1
        /* Call registered callback*/
        hfdcan->RxFifo0Callback(hfdcan, RxFifo0ITs);
    #else
        /* Rx FIFO 0 Callback */
        HAL_FDCAN_RxFifo0Callback(hfdcan, RxFifo0ITs);
    #endif /* USE_HAL_FDCAN_REGISTER_CALLBACKS */
    }
```

### 2.4中断接收函数
```c++

我们在中断中使用串口打印can收到的消息

// 中断回调函数
void HAL_FDCAN_RxFifo0Callback(FDCAN_HandleTypeDef *hfdcan, uint32_t RxFifo0ITs)
{
	uint8_t  data[8];
  // 检查是否是需要的中断回调
	if((RxFifo0ITs & FDCAN_IT_RX_FIFO0_NEW_MESSAGE) != RESET)
  {
		//获取消息头和数据。注意消息头结构体中包含过滤器编号
		if (HAL_FDCAN_GetRxMessage(hfdcan, FDCAN_RX_FIFO0, &fdcan1_RxHeader, data) != HAL_OK)
		{
			Error_Handler();
		}
			printf("--->Data Receieve!\r\n");
			printf("RxMessage.StdId is %#x\r\n",  fdcan1_RxHeader.Identifier);//打印编号Id
			printf("data[0] is 0x%02x\r\n", data[0]);
			printf("data[1] is 0x%02x\r\n", data[1]);
			printf("data[2] is 0x%02x\r\n", data[2]);
			printf("data[3] is 0x%02x\r\n", data[3]);
			printf("data[4] is 0x%02x\r\n", data[4]);
			printf("data[5] is 0x%02x\r\n", data[5]);
			printf("data[6] is 0x%02x\r\n", data[6]);
			printf("data[7] is 0x%02x\r\n", data[7]);		
			printf("<---\r\n");
	}
}
```

在`FDCAN1_IT0_IRQHandler();`中我们看到了，只要FIFO0中断被配置了，不管是FDCAN_IT_RX_FIFO0_MESSAGE_LOST 还是 FDCAN_IT_RX_FIFO0_FULL或者是    FDCAN_IT_RX_FIFO0_NEW_MESSAGE，都会调用这个函数。

但是，很有意义的是，HAL将到底是哪个类型的中断作为参数RxFifo0ITs传递给了这个回调函数，所以我们需要判断一下到底是哪个中断类型。用与运算：

	if((RxFifo0ITs & FDCAN_IT_RX_FIFO0_NEW_MESSAGE) != RESET)
    {······}

然后，就可以获取消息报文了，消息报文分两部分，一个是消息头、一个是数据

```c++
    //获取消息头和数据。注意消息头结构体中包含过滤器编号
    if (HAL_FDCAN_GetRxMessage(hfdcan, FDCAN_RX_FIFO0, &fdcan1_RxHeader, data) != HAL_OK)
    {
        Error_Handler();
    }
```

消息头结构体中的信息还是很丰富的，尤其对于can-FD增加了很多数据，当然对于经典can用不到。比较有用的是：

* Identifier 帧ID
* IdType 帧ID的类型
* RxFrameType 帧的类型
* DataLength 数据长度
* FilterIndex，用于判断是哪一个过滤器放进来的消息，过滤器编号方式见上一篇f103经典can的配置。


**注：** 我感觉对于一个FIFOx，应该是可以同时配置多种中断的，毕竟`__HAL_FDCAN_ENABLE_IT(hfdcan, ActiveITs);`就是与运算，因该多次调用`HAL_FDCAN_ActivateNotification()`可以使能多个FIFO0的的中断。

还有一些奇怪的函数，等以后再研究，例如HAL_FDCAN_ConfigGlobalFilter()，似乎是可以将所有的消息全都放进来，然后在接收到的消息的消息头中IsFilterMatchingFrame判断是不是这个函数放进来的。



### 2.5 发送

发送就比较简单了，同样在 fdcan.c的`/* USER CODE BEGIN 1 */`里面定义发送数据的函数

```c++
//msg:数据指针,最大为8个字节.
//返回值:0,成功;
//其他,失败;
uint8_t FDCAN1_Send_Msg(uint8_t* msg)
{	
		
  fdcan1_TxHeader.Identifier = 0x001;							//帧ID
  fdcan1_TxHeader.IdType = FDCAN_STANDARD_ID;			//标准帧or扩展帧
  fdcan1_TxHeader.TxFrameType = FDCAN_DATA_FRAME;	//帧类型，这里是数据帧
  fdcan1_TxHeader.DataLength = FDCAN_DLC_BYTES_8;	//数据段长度，8 bytes
  fdcan1_TxHeader.ErrorStateIndicator = FDCAN_ESI_ACTIVE;	//错误状态指示位，canFD相关的这里不用管
  fdcan1_TxHeader.BitRateSwitch = FDCAN_BRS_OFF;					//波特率切换，canFD相关的这里设置为关闭（不用管应该也行）
  fdcan1_TxHeader.FDFormat = FDCAN_CLASSIC_CAN;						//设置帧的格式为标准can还是FDcan
  fdcan1_TxHeader.TxEventFifoControl = FDCAN_NO_TX_EVENTS;		// 无发送事件
  fdcan1_TxHeader.MessageMarker = 0x00; //由于网上借鉴该函数，我也不太明白为什么是0x52，不过我测试改成0好像也没问题                    
    
  if(HAL_FDCAN_AddMessageToTxFifoQ(&hfdcan1,&fdcan1_TxHeader,msg)!=HAL_OK) return 1;//发送
	return 0;	
}
```
要注意的主要有以下几点：

* 设置为经典can模式：

    fdcan1_TxHeader.FDFormat = FDCAN_CLASSIC_CAN;


* 注意这里对于经典模式下的can，数据长度fdcan1_TxHeader.DataLength最高为8个字节，可设置范围为FDCAN_DLC_BYTES_2~FDCAN_DLC_BYTES_8

这个参数是FDcan的，配置是否使能数据段切换波特率，对于经典can没用，默认就是0，0就是OFF，所以保持默认不用改或者按照这里配置也行，反正没用

    fdcan1_TxHeader.BitRateSwitch = FDCAN_BRS_OFF

这些是配置发送事件的，直接设置为无（默认也是，所以不配置也行），暂时不懂，以后用到再说：

  fdcan1_TxHeader.TxEventFifoControl = FDCAN_NO_TX_EVENTS;  // 无发送事件
  fdcan1_TxHeader.MessageMarker = 0x00; 

### 2.6 main函数

由于我们配置了工作模式为Internal loop back模式，也就是自己拉出来的再吃进去，所以发出的消息可以被收到直接进中断，然后在CAN中断里打印收到的消息，这样验证CAN收发是否配置正确

```c++
  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
		HAL_Delay(200);
		
		user_main_printf("stm32");
		char data[5] = "hello"; 
		FDCAN1_Send_Msg((uint8_t*)(data));
  }
  /* USER CODE END 3 */
```

## 查询方式接收——不常用

原理是直接用FDCAN1_Receive_Msg()接收消息

```c++
uint8_t FDCAN1_Receive_Msg(uint8_t *buf, uint16_t *Identifier,uint16_t *len)
{	
    
  if(HAL_FDCAN_GetRxMessage(&hfdcan1,FDCAN_RX_FIFO0,&fdcan1_RxHeader,buf)!=HAL_OK)return 0;//接收数据
  *Identifier = fdcan1_RxHeader.Identifier;
  *len=fdcan1_RxHeader.DataLength>>16;
  return fdcan1_RxHeader.DataLength>>16;	
}
```