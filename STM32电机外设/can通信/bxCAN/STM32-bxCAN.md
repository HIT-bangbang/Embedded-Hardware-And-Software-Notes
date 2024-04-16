# 接收和发送流程 以stm32F1为例（使用bxCAN的芯片）

STM32G4系列和H7的can均为FD-can（支持FDcan，兼容经典can）
stm32f1和F4的叫做bxCAN（Basic Extended CAN）

**Mailbox邮箱**：软件与硬件交互信息的接口，我认为是用于发送报文的发送调度器，因为stm32并没有使用“接收邮箱”这一概念，而接收用的是FIFO

**接收**：

    数据帧-->过滤器-->接收FIFO-->读出

**发送**：

    数据帧-->发送邮箱-->发送

## 一 、发送

”发送邮箱“是用于CAN总线数据发送的，每个CAN控制器总共有3个发送邮箱，每个邮箱存储一条待发送报文，邮箱之间存在优先级关系（可配置优先级比较规则）。优先级越高表示其里面的数据会被优先发送。数据在发送前被送到空闲的发送邮箱，然后根据优先级依次发送。

总结：“发送邮箱有3个，且每个邮箱只能装一个报文”。

一个发送邮箱其实是由四个寄存器构成，每个发送邮箱中包含有标识符寄存器 CAN_TIxR、数据长度控制寄存器 CAN_TDTxR 及 2 个数据寄存器（高位和低位） CAN_TDLxR、CAN_TDHxR

| 寄存器 | 功能 |
|-------|-------|
| 标识符寄存器 CAN_TIxR | 存储待发送报文的ID、扩展ID、IDE位以及RTR位 |
| 数据长度控制寄存器 CAN_TDTxR | 存储待发送报文的DLC段 |
| 低位数据寄存器 CAN_TDLxR | 存储待发送报文数据段的Data0-Data3 |
| 高位数据寄存器 CAN_TDHxR | 储待发送报文数据段的Data4-Data7 |


### 1.1发送邮箱状态切换

<image src="1.jpg">


发送邮箱共有四种状态，空状态（empty），挂号状态（pending），预定发送状态（scheduled），发送状态（transmit）

发送报文的流程为：

应用程序选择1个**空状态**的邮箱；

设置标识符，数据长度和待发送数据；然后对CAN_TIxR寄存器的TXRQ位置’1’，来请求发送。TXRQ位置’1’后，邮箱就不再是空邮箱；而一旦邮箱不再为空，软件对邮箱寄存器就不再有写的权限。

TXRQ位置1后，邮箱马上进入**挂号状态**，并等待成为最高优先级的邮箱，优先级的评定方式取决于发送优先级配置，参见**发送优先级**。

一旦邮箱成为当前最高优先级的邮箱，其状态就变为**预定发送状态**，并等待CAN总线空闲。

一旦CAN总线进入空闲状态，预定发送邮箱中的报文就马上被发送(进入**发送状态**)。一旦邮箱中的报文被成功发送后，它马上变为空邮箱（**空状态**）；硬件相应地对CAN_TSR寄存器的RQCP和TXOK位置1，来表明一次成功发送。


如果发送失败，由于仲裁引起的就对CAN_TSR寄存器的ALST位置’1’，由于发送错误引起的就对TERR位置’1’。

总结起来就是：

**空状态**-->写入数据-->**挂号状态**-->获得最高优先级-->**预定发送状态**-->CAN总线空闲-->**发送状态**-->发送完成-->空状态

注：Can总线何时是空闲的：

    当总线连续表现为11个位的隐形电平(高电平)，则总线为空闲状态。但是Can总线不是说5个相同位后就会有一个反转位码，那是发生在发送数据时。总线空闲时没有发送数据,发送数据时不会出现这么多隐性位。

### 1.2发送优先级

STM32共有三个CAN发送邮箱，在检测到总线空闲时才可以发送邮箱内的数据。

发送邮箱中有可能同时存在多个需要发送的报文，一旦出现这种情况，那么发送邮箱中的多个报文又将是谁先发送谁后发送呢？有两种模式：ID模式和FIFO模式。

ID模式由报文的ID值决定，即ID值越小，优先级越高。当有超过1个发送邮箱在挂号时，发送顺序由三个邮箱中报文的标识符决定。根据CAN协议，标识符数值最低的报文具有最高的优先级。如果标识符的值相等，那么邮箱号小的报文先被发送。此模式通过对CAN主控寄存器CAN_MCR的TXFP位清0来设置。

FIFO模式，顾名思义，即为消息队列方式，报文先被写入邮箱的，就先发送该消文。通过对CAN_MCR寄存器（CAN主控寄存器）的TXFP位置’1’，可以把发送邮箱配置为发送FIFO。在该模式下，发送的优先级由发送请求次序决定。该模式对分段发送很有用。

### 1.3取消发送
发送邮箱中待发送的报文在正常发送成功之前也可以中途取消，通过对CAN_TSR寄存器的ABRQ位置’1’，可以中止发送请求。

当发送邮箱处于挂号或预定状态时：发送请求马上就被中止了。

当发送邮箱处于发送状态时，中止请求可能导致2种结果：

        1：如果邮箱中的报文被成功发送，那么邮箱变为空邮箱，并且CAN_TSR寄存器（CAN发送状态寄存器）的TXOK位被硬件置’1’；
        2：如果邮箱中的报文发送失败了，那么邮箱变为预定状态，然后发送请求被中止，邮箱变为空邮箱且TXOK位被硬件清’0’。

因此，不管如何，一旦取消发送，那么在发送操作结束后，邮箱都会变为空邮箱。


### 1.4自动重传模式

报文发送有可能会发送失败，有可能因为总线竞争仲裁失败，也有可能是其它错误。

可选自动重传该帧报文，也可以关闭自动重传，关闭自动重传后，出现失败后该帧报文丢弃。

根据我的经验来看，这一个还是关闭比较好，因为相比较于丢失几帧数据的代价，重传导致的延迟会导致更加严重的后果。


## 二、接收

stm32的hal库里并没有使用RxMailbox这一说法。接收使用的是FIFO

”接收FIFO“用于CAN总线数据接收，在接收数据端会有一个过滤器处于”接收FIFO“的前面，只有标识符符合的报文才会顺利通过过滤器，被加入到”接收FIFO“当中。

接收邮箱有2个(FIFO0、FIFO1)，每个FIFO的大小为三个报文，即每一个接收FIFO可以暂存三个报文，如下图所示。但读取时只能读到最先加入FIFO的报文，等这个读完之后，才能读下一个报文”。

<image src="2.png">


### 2.1过滤器组

为什么叫过滤器“组”呢？因为一个过滤器组里面可能有几个过滤器。例如配置为16位+列表模式后，这个过滤器组里面其实就有了四个过滤器，而配置为32位+屏蔽模式，那么这个过滤器组就只有一个过滤器。

每个过滤器组由两个32位寄存器构成CAN_FxR1,CAN_FxR2。例如stm32F103只有一个can控制器，该控制器有13个过滤器组，每个过滤器由CAN_FxR1（x=1-13）和CAN_FxR2（x=1-13）组成。

每个过滤器组都可以配置位宽和模式，其实排列组合一共就四种。

位宽有两种：32位和16位。

过滤器组模式两种：列表模式和屏蔽模式：

标识符列表模式：只接收某几个指定ID的消息。它把要接收报文的ID列成一个表，要求报文ID与列表中的某一ID完全相同才可以接收，可以理解为做了一个白名单，只有在白名单上面的ID才会被接收。

屏蔽（掩码）模式，它把可接收报文ID的某几位作为列表，这几位被称为掩码，可以把它理解成关键字搜索，只要掩码(关键字)相同，就符合要求，报文就会被保存到接收 FIFO。

<image src="3.png">

图中第一种：

该过滤器组的两个32位寄存器分别被用作存放一个32位的ID和存放一个32位的屏蔽掩码。-->一个过滤器

图中第二种：

该过滤器组的两个32位寄存器分别被用作存放一个32位的ID和另一个32位的ID。这样的话，符合这两个ID的报文就都可以通过该过滤器到达FIFO（前提是该FIFO使用的就是这个过滤器）-->两个过滤器

图中第三种：

两个32位寄存器分别被拆成了两个16位。这样对于CAN_FxR1，它的低16位就可以用来放ID，高16位用来放屏蔽掩码。对于CAN_FxR2同理。-->两个过滤器

图中第四种：

两个32位寄存器分别被拆成了两个16位。这样一共就有了四个16位，可以存放四个16位的ID。can总线上的符合这4个ID的报文就都可以通过该过滤器。-->四个过滤器

这主要是为了满足扩展帧和标准帧ID长度的差异，以及使用方便的需求。例如对于标准帧（11位ID），一般使用16位位宽，这样可以获得更多可选的屏蔽或者列表模式，没必要浪费使用32位。对于扩展帧（29位ID），就没办法使用16位的位宽进行筛选过滤了，需要使用32位的位宽来配置ID列表或者屏蔽掩码。

### 2.2FIFO与过滤器组的绑定

一个FIFO前可以绑定多个过滤器组，一个过滤器组只能绑定到一个FIFO前面。

STM32的每个CAN有两个FIFO，分别是FIFO0和FIFO1。为了便于区分，下面FIFO0写作FIFO_0，FIFO1写作FIFO_1。

每组过滤器组必须关联且只能关联一个FIFO。复位默认都关联到FIFO_0。

    所谓“关联”是指假如收到的报文能从某个过滤器通过，那么该报文会被存到该过滤器相连的FIFO。

从另一方面来说，每个FIFO都关联了一串的过滤器组，例如使stm32F4的两个FIFO刚好瓜分了所有的过滤器组。

    每当收到一个报文，CAN就将这个报文先与FIFO_0关联的过滤器依次比较，如果有任意一个通过了，就将此报文放入FIFO_0中。如果不匹配，再将报文与FIFO_1关联的过滤器依次比较，如果有任意一个被匹配，该报文就放入FIFO_1中。如果还是不匹配，此报文就被丢弃。

    每个FIFO的所有过滤器都是并联的，只要通过了其中任何一个过滤器，该报文就会被存到这个FIFO里。

如果一个报文既符合FIFO_0的规定，又符合FIFO_1的规定，显然，根据操作顺序，它会被放到FIFO_0中。

每个FIFO中只有激活了的过滤器才起作用，换句话说，如果一个FIFO有20个过滤器，但是只激话了5个，那么比较报文时，只拿这5个过滤器作比较。

    一般要用到某个过滤器时，在初始化阶段就直接将它激活。

    需要注意的是，每个FIFO必须至少激活一个过滤器，它才有可能收到报文。如果一个过滤器都没有激活，那么是所有报文都报废的。


报文通过过滤器被加入到FIFO中时，具体是从哪个哪个过滤器来的也会被记录下来一起存入接收邮箱，过滤器匹配序号存放在CAN_RDTxR寄存器的FMI域中。这样就可以使用该编号确定这条报文的类型，而不用在程序中对报文的ID进行判断，浪费计算资源了。

但是这里有一个问题，这么多滤波器，先跟谁比较呢？

    1、位宽为32位的过滤器，优先级高于位宽为16位的过滤器
    2、对于位宽相同的过滤器，标识符列表模式的优先级高于屏蔽位模式
    2、位宽和模式都相同的过滤器，优先级由过滤器编号决定，过滤器编号小的优先级高。注意是过滤器编号，而不是过滤器组，因为一个过滤器组中可能有多个过滤器，这样并不能区分到底是什么类型的消息。

注意这里的过滤器编号是每个FIFO各自给绑定到自己的滤波器安排的。STM32使用以下规则对过滤器编号：

(1) FIFO_0和FIFO_1的过滤器分别独立编号，均从0开始按顺序编号。

(2) 所有关联同一个FIFO的过滤器，不管有没有被激活，均统一进行编号。

(3) 编号从0开始，按过**滤器组**的编号从小到大，按顺序排列。

(4) 在同一过滤器组内，按寄存器从小到大编号。FxR1配置的过滤器编号小，FxR2配置的过滤器编号大。

(5) 同一个寄存器内，按位序从小到大编号。[15-0]位配置的过滤器编号小，[31-16]位配置的过滤器编号大。


例如图中，首先按照寄存器组编号来排序。对于配置为32位+列表模式的过滤器组0，其中有两个过滤器，FxR1配置成的过滤器编较小的编号0，FxR2配置成的过滤器编较大的编号1。对于配置为16位+列表模式的过滤器组3，其中有四个过滤器，首先按照规则(4)，FxR1配置成的过滤器使用较小的编号3和4，FxR2配置成的过滤器使用较大的编号5和6。之后，按照规则（5）FxR1也被拆成了两个16位过滤器，用FxR1低16位配置成的过滤器就编号为3，高16位的编号为4。对于FxR2同理，低16位过滤器编号5，高16位过滤器编号6。

<image src="4.png">


一般的，如果不想用复杂的过滤功能，FIFO可以只激活一组过滤器组，且将它设置成32位的屏蔽位模式，两个标准值寄存器FxR1、FxR2都设置成0（其实只要屏蔽位FxR2全部设成零），这样所有报文均能通过。

### 2.3一个FIFO配置多个滤波器

```c++
void CANFilter_Config(void)
{
    CAN_FilterTypeDef  sFilterConfig_1; // 过滤器1
    CAN_FilterTypeDef  sFilterConfig_2; // 过滤器2

    sFilterConfig_1.FilterBank = 0;                       //CAN过滤器编号，范围0-27
    sFilterConfig_1.FilterMode = CAN_FILTERMODE_IDMASK;   //CAN过滤器模式，掩码模式或列表模式
    sFilterConfig_1.FilterScale = CAN_FILTERSCALE_32BIT;  //CAN过滤器尺度，16位或32位
    sFilterConfig_1.FilterIdHigh = 0x000 << 5;		//32位下，存储要过滤ID的高16位
    sFilterConfig_1.FilterIdLow = 0x0000;					//32位下，存储要过滤ID的低16位
    sFilterConfig_1.FilterMaskIdHigh = 0x0000;		//掩码模式下，存储的是掩码
    sFilterConfig_1.FilterMaskIdLow = 0x0000;
    sFilterConfig_1.FilterFIFOAssignment = 0;			//报文通过过滤器的匹配后，存储到哪个FIFO
    sFilterConfig_1.FilterActivation = ENABLE;    //激活过滤器
    sFilterConfig_1.SlaveStartFilterBank = 0;
    
    sFilterConfig_2 = sFilterConfig_1;
    sFilterConfig_2.FilterBank = 1;                       //CAN过滤器编号，范围0-27
    ......(省略其他配置过程)

    // 配置接收过滤器1
    if (HAL_CAN_ConfigFilter(&hcan, &sFilterConfig_1) != HAL_OK) 
    {
        Error_Handler();
    }
    else
    {
        printf("HAL_CAN_ConfigFilter(&hcan, &sFilterConfig) is HAL_OK\r\n"); 
    }
    // 配置接收过滤器2
    if (HAL_CAN_ConfigFilter(&hcan, &sFilterConfig_2) != HAL_OK) 
    {
        Error_Handler();
    }
    else
    {
        printf("HAL_CAN_ConfigFilter(&hcan, &sFilterConfig) is HAL_OK\r\n"); 
    }
}

```
### 2.4使用两个FIFO

注意在CubeMx中打开对应的接收中断。

对于stm32f103，FIFO_0的接收中断和USB共用一个，`USB_LP_CAN1_RX0_IRQHandler`，而FIFO_1的接收中断为`CAN1_RX1_IRQHandler`

对于F4，就很明了了，can1的FIFO_0的接收中断为`CAN1_RX0_IRQHandler`，can2的FIFO_1的接收中断为`CAN1_RX1_IRQHandler`

依次使能两个FIFO的接收中断：
```c++
// 使能FIFO0接收中断
if (HAL_CAN_ActivateNotification(&hcan, CAN_IT_RX_FIFO0_MSG_PENDING) !=  HAL_OK){
        Error_Handler();
}
else{ 
    printf("HAL_CAN_ActivateNotification(&hcan, CAN_IT_RX_FIFO0_MSG_PENDING) is HAL_OK\r\n"); }	

// 使能FIFO0接收中断
if (HAL_CAN_ActivateNotification(&hcan, CAN_IT_RX_FIFO0_MSG_PENDING) !=  HAL_OK){
        Error_Handler();
}
else{ 
    printf("HAL_CAN_ActivateNotification(&hcan, CAN_IT_RX_FIFO0_MSG_PENDING) is HAL_OK\r\n"); }	

```

## 2.5 接收中断函数跳转逻辑：

1、当接收中断被触发时，程序进入`USB_LP_CAN1_RX0_IRQHandler`。（stm32f1xx_it.c）

2.`USB_LP_CAN1_RX0_IRQHandler`内调用`HAL_CAN_IRQHandler`。（stm32f1xx_it.c）

3.`HAL_CAN_IRQHandler`内调用`HAL_CAN_RxFifo0MsgPendingCallback`。（stm32f1xx_hal_can.c）

4.`HAL_CAN_IRQHandler`调用哪一些回调函数是我们用`HAL_CAN_ActivateNotification(&hcan, CAN_IT_RX_FIFO0_MSG_PENDING)`进行配置的，其实就是将`HAL_CAN_IRQHandler`里面的某一个`if`判断置位。

5.`HAL_CAN_RxFifo0MsgPendingCallback`回调函数里面，我们就可以调用`HAL_CAN_GetRxMessage`获取接收到的报文。

6.`HAL_CAN_GetRxMessage`函数将(CAN_RIxR)接收到的ID、RTR、DLC、过滤器匹配序号、时间戳存入`CAN_RxHeaderTypeDef`类型的结构体中

至于接收之前的使能标志位，接收之后释放FIFO就不需要我们关心了。

## 2.6 数据头

```c++
// 接收数据头
typedef struct
{
  uint32_t StdId;    /*!< Specifies the standard identifier.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 0x7FF. */

  uint32_t ExtId;    /*!< Specifies the extended identifier.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 0x1FFFFFFF. */

  uint32_t IDE;      /*!< Specifies the type of identifier for the message that will be transmitted.
                          This parameter can be a value of @ref CAN_identifier_type */

  uint32_t RTR;      /*!< Specifies the type of frame for the message that will be transmitted.
                          This parameter can be a value of @ref CAN_remote_transmission_request */

  uint32_t DLC;      /*!< Specifies the length of the frame that will be transmitted.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 8. */

  uint32_t Timestamp; /*!< Specifies the timestamp counter value captured on start of frame reception.
                          @note: Time Triggered Communication Mode must be enabled.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 0xFFFF. */

  uint32_t FilterMatchIndex; /*!< Specifies the index of matching acceptance filter element.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 0xFF. */

} CAN_RxHeaderTypeDef;

// 发送数据头
typedef struct
{
  uint32_t StdId;    /*!< Specifies the standard identifier.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 0x7FF. */

  uint32_t ExtId;    /*!< Specifies the extended identifier.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 0x1FFFFFFF. */

  uint32_t IDE;      /*!< Specifies the type of identifier for the message that will be transmitted.
                          This parameter can be a value of @ref CAN_identifier_type */

  uint32_t RTR;      /*!< Specifies the type of frame for the message that will be transmitted.
                          This parameter can be a value of @ref CAN_remote_transmission_request */

  uint32_t DLC;      /*!< Specifies the length of the frame that will be transmitted.
                          This parameter must be a number between Min_Data = 0 and Max_Data = 8. */

  FunctionalState TransmitGlobalTime; /*!< Specifies whether the timestamp counter value captured on start
                          of frame transmission, is sent in DATA6 and DATA7 replacing pData[6] and pData[7].
                          @note: Time Triggered Communication Mode must be enabled.
                          @note: DLC must be programmed as 8 bytes, in order these 2 bytes are sent.
                          This parameter can be set to ENABLE or DISABLE. */
} CAN_TxHeaderTypeDef;







```



## 2.7 CRC校验
