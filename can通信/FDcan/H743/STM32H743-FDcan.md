# H743 FD-CAN

## 1、与G4系列的FD-CAN的主要区别

### 更灵活的Message RAM

G4的Message RAM一共有212 words的空间（所有FD-can共用），RAM中各个区域的大小、地址已经分配好，并且是固定的

H743 FD-CAN的Message RAM有2560 words（两个FD-can共用），RAM中各个区域的大小、地址是浮动的，如下图所示：

<image src="Message-RAM.png">

每个区域都需要自己配置大小，这个图只是个示意，实际物理上存储的顺序、大小都不一定和图上一样。

一个element也就是一个元素，可以是一个滤波器或者是FIFO中存的一个报文等等。一个element不一定占多大的空间，这个根据element的类型来看。

* 11-bit filter ：每个11位标准帧ID过滤器占用1个word，最多可以使用128个，也就是占128 words
* 29-bit filter ： 每个29位扩展帧ID过滤器占用2个word，最多可以使用64个，也就是占128 words
* Rx FIFO 0 ： 最多可以存64个报文，一个报文到底需要多少word与数据段的长度有关，如果数据段长度为8 bytes 那么一个这样的报文在FIFO中需要4个word（具体可以看Keil源码`RxFifo0ElmtSize`，占用的 $ num of word = 数据段字节数*8 /32 + 2 $ ）。一个报文的数据段最高是64bytes，这时存一个报文需要18 words，总共可以存64个，最多就是 $64*18=1152$
* Rx FIFO 1 : 同Rx FIFO 0
* Rx buffer : 同Rx FIFO 0
* Tx event FIFO ：最多存32个发送事件，总共最多占用64words
* Tx buffers :  0-32 elements / 0-576 words
* Trigger memory:  0-32 elements / 0-576 words


**注意**，如果全部把上面的配置满，$128 + 128 + 1152 + 1152 + 1152 + 64 + 576 + 128 = 4480$ 就会发现此时早就远远超过这一块RAM的存储空间了，所以配置的时候要注意是不是已经超了，而且要注意两个FD-can是共用这一块RAM区域的，他们的FIFO，滤波器等等都是存在这里面。

**注意** 系统不会对 Message RAM 配置进行检查，一定注意是否会发生溢出或者越界的风险。

### Rx buffer
似乎是和Rx FIFO差不多的东西，过滤器既可以绑定到FIFO0或者FIFO1中的一个，也可以绑定到Rx buffer，

过滤器绑定到Rx buffer以后，通过了的消息直接存到buffer里面。暂时不用，设置为0

These filters can be assigned to Rx buffer, Rx FIFO 0 or Rx FIFO 1

### Trigger memory

以后再研究

## 2、CubeMX配置


<image src="CubeMx.png">

大部分参数和G4的配置一样，下面主要说一下不同的地方。

* 没有了Clock Divider：时钟分频
* Message Ram Offset：选择这个CAN在RAM中的存储空间首地址相对于RAM0地址的偏置。
* Rx Fifo0 Elmts Nbr： Rx Fifo0的element的数量，也就是最多存几条报文
* Rx Fifo0 Elmt Size： Rx Fifo0中报文的长度（其实选项里面是选择数据段的长度，Cubemx自动转换成Elmt Size）
* Rx Fifo0 Elmts Nbr 和 Rx Fifo0 Elmt Size同上
* Rx Buffers Nbr 和 Rx Buffer Size 同上
* Tx Events Nbr
* Tx Buffers Nbr
* Tx Fifo Queue Elmts Nbr：对应RAM中Tx buffers的element数量（发送缓冲区中报文数量）
* Tx Fifo Queue Mode：和G4的一样，用来配置Fifo模式还是Queue模式，与发送的优先级有关
* Tx Elmt Size：报文的长度（其实选项里面是选择数据段的长度，Cubemx自动转换成Elmt Size）

着重说一下Message Ram Offset这个参数，主要是因为我们有两个FD-CAN，而且这两个FD-CAN共用一块RAM。在配置时，就需要将这两个FD-CAN使用到的内存区域分开，到底谁从哪里开始，到哪里结束，接下来时谁用，避免冲突。例如我们首先配置FD-CAN1的RAM区域，之后要配置FD-CAN2时，显然要从FD-CAN1的RAM区域的末端开始接上。

Message Ram Offset就是用来配置FD-CAN1和FD-CAN2在RAM中的起始地址偏置。

错误做法：在STM32CubeMX上，FDCAN1与FDCAN2的Message RAM Offset都设置为0。这样的话，FDCAN1的消息RAM与FDCAN2的消息RAM重叠在一起了，如图，显然是错误的。

<image src="wrong-offset.png">

hal库中`FDCAN_CalcultateRamBlockAddresses()`函数完成了对各个区域地址的计算，而且很贴心的帮忙做了一个EndAddress：

```c++
hfdcan->msgRam.StandardFilterSA = SRAMCAN_BASE + (hfdcan->Init.MessageRAMOffset * 4U);
hfdcan->msgRam.ExtendedFilterSA = hfdcan->msgRam.StandardFilterSA + (hfdcan->Init.StdFiltersNbr * 4U);
hfdcan->msgRam.RxFIFO0SA = hfdcan->msgRam.ExtendedFilterSA + (hfdcan->Init.ExtFiltersNbr * 2U * 4U);
hfdcan->msgRam.RxFIFO1SA = hfdcan->msgRam.RxFIFO0SA + (hfdcan->Init.RxFifo0ElmtsNbr * hfdcan->Init.RxFifo0ElmtSize * 4U);
hfdcan->msgRam.RxBufferSA = hfdcan->msgRam.RxFIFO1SA + (hfdcan->Init.RxFifo1ElmtsNbr * hfdcan->Init.RxFifo1ElmtSize * 4U);
hfdcan->msgRam.TxEventFIFOSA = hfdcan->msgRam.RxBufferSA + (hfdcan->Init.RxBuffersNbr * hfdcan->Init.RxBufferSize * 4U);
hfdcan->msgRam.TxBufferSA = hfdcan->msgRam.TxEventFIFOSA + (hfdcan->Init.TxEventsNbr * 2U * 4U);
hfdcan->msgRam.TxFIFOQSA = hfdcan->msgRam.TxBufferSA + (hfdcan->Init.TxBuffersNbr * hfdcan->Init.TxElmtSize * 4U);

hfdcan->msgRam.EndAddress = hfdcan->msgRam.TxFIFOQSA + (hfdcan->Init.TxFifoQueueElmtsNbr * hfdcan->Init.TxElmtSize * 4U);
```

SRAMCAN_BASE就是Message RAM的起始地址，offset相对于这一地址。

MessageRAMOffset在FDCAN1的时候设置为0，在配置FDCAN2的时候需要根据配置来进行计算，hal库很贴心的帮忙做了一个EndAddress，这样 FDCAN2 的 MessageRAMOffset 可以直接设置为 FDCAN1 的 EndAddress - SRAMCAN_BASE。（当然应该可以写的稍微大一点，不紧挨着CAN1应该也行，就是又浪费又没必要）我们可以在Cubemx中随便写0，然后在Keil中手动修改如下：

```c++
  hfdcan2.Init.MessageRAMOffset = hfdcan1.msgRam.EndAddress - SRAMCAN_BASE;

```

这样两个CAN使用的RAM空间才是头接尾，互不重叠的。

<image src="right-offset.png">

## 3、Keil中的配置与G431相同