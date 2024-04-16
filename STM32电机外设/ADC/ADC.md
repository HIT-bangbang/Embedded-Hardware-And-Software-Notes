# 一、STM32 ADC 概况和注意事项

* 阻塞（查询）模式
* 中断模式
* DMA模式

* 分辨率 最大为12bit（H7为16bit）可以配置为10,8,6 bit
* 数据对齐方式可以配置为 左对齐 或 右对齐
* 多个ADC的协同模式：为独立或多重转换模式
* 单个ADC通道采样模式： 单次、连续、扫描
* 时钟 <= 36Mhz （不同的芯片有不同的要求）
* 两次采样间隔为5-10CLK
* 16个规则组和4个注入组
* 内置温度、Vref和Vbat通道
* 允许定时器和外部源触发，在电机控制**电流采样**时有用
* ADC看门狗，可以设置触发看门狗的高低阈值

## 1.1 采样时间：

$$T_{ADC} = T_{sam} + 12T_{ADCCLK}$$
$$T_{sam} = 3 ~ 480T_{ADCCLK}$$

$T_{sam}$可以设置，采样时间越长，结果越准确

## 1.2 单次（非扫描）、连续、间断（不连续）、扫描

单次、连续、扫描是针对于某一个ADC的通道而言的。单次转换在该ADC完成转换一次数据就停止，要再次触发转换才可以，连续模式则是无限循环这个过程，不需要再次触发。

有时我们可能需要对多个ADC通道进行分组转换，组与组之间希望有可调的时间间隔。例如，我们想先转换前2个通道chx和chy，再转换中间2个通道chm和chn，之后转换最后的2个通道chi和chj。

扫描模式开启后，每一次使能ADCx，会依次扫描在 ADC_SQRx 寄存器(对于规则通道)或 ADC_JSQR 寄存器(对于注入通道)中选择的所有通道。其顺序也是由这两个寄存器决定的。

扫描模式关闭后，即使ADC_SQRx或ADC_JSQR选中了多个通道，每次使能ADCx也只会对选中的第一个通道做转换。

以ADC1为例，规则组通道的顺序为CH0,CH1,CH2,CH3:

* 不启动SCAN模式，在单次转换模式下启动ADC1，则：

    1.开始转换CH1（ADC_SQR的第一通道）

    转换完成后停止，等待ADC的下一次启动，继续从第一步开始转换

* 不启动SCAN模式，在连续转换模式下，启动ADC1，则：

    1.开始转换CH0（ADC_SQR的第一通道）

    转换完成后回到第一步。

* 启动SCAN模式，在单次转换模式下，启动ADC1，则：

    1.开始转换CH0、

    2.转换完成后自动开始转换CH1

    3.转换完成后自动开始转换CH2

    4.转换完成后自动开始转换CH3

    5.转换完成后停止，等待ADC的下一次启动下一次ADC启动后从第一步开始转换

* 启动SCAN模式，在连续转换模式下，启动ADC1，则：

    1.开始转换CH0、

    2.转换完成后自动开始转换CH1

    3.转换完成后自动开始转换CH2

    4.转换完成后自动开始转换CH3

    5.转换完成后返回第一步

## 1.3 一些模式的解释

`Scan Conversion Mode`(扫描模式)：采用多通道必须开启，比如开了ch0，ch1，ch4，ch5。Ch0转换完以后就会自动转换通道1,4,5直到0、1、4、5都转换完就停止。但是这种连续性并不是不能被打断。这就引入了间断模式.

`Discontinuous Conversion Mode`，可以说是对扫描模式的一种补充。它可以把0,1,4,5这四个通道进行分组。可以分成0,1一组，4,5一组。也可以每个通道配置为一组。这样每一组转换之前都需要先触发一次。Discontinuous Conversion Mode 使能后number of Discontinous Conversions是配置间断组每个组有几个通道的，这里必须配置为1（否则在获取ad值得时候只能读取到每个间断组最后一个通道）。

`Continuous Conversion Mode`(连续转换模式)：Stm32 ADC的单次模式和连续模式。这两中模式的概念是相对应的。这里的单次模式并不是指一个通道，而是整个组（规则组）。假如你同时开了ch0，ch1，ch4，ch5这四个通道。单次模式转换模式下会把这四个通道采集一遍就停止了。而连续模式就是这四个通道转换完以后再循环过来再从ch0开始（连续转换模式一般情况下配合扫描模式工作）。

`DMA Continuous Requests`(DMA连续请求模式)：**在main.c中使用HAL_ADC_START_DMA(&hadc1,(xint32_t *)butter 100)这个语句，使用ADC1和DMA，数据放入buffer数组内，放100个数据。如果DMA连续请求模式失能，这句语句传输完100个数据后自动关闭ADC1和DMA；反之使能后，语句执行完后又会连续从头开始传输数据，即buffer数组中的值一直在更新。

**开启扫描模式后 一般要搭配DMA或中断功能才能实现ADC的数据处理**

**使用多通道都需要要开启扫描模式**

在adc+dma多通道采集数据时可能会出现数据错位的情况（数据不按照设置的Rank优先级顺序存放到数组中）。可能原因是：采样周期小，而数据长度短，使得转换过快，DMA还没读取就被覆盖了，需要把采样周期变长。


## 1.4 独立/多重模式

STM32有三个ADC，每个ADC又有自己多个通道。独立/多重模式是针对于这三个ADC而言的。

独立模式下，多个ADC不能协同工作，每个ADC接口独立工作。

多重模式下，多个ADC可以协同工作，例如多重ADC采样同步模式和交替模式。

所以如果不需要ADC同步或者只是用了一个ADC的时候，就应该设成独立模式了。



# 二、查询模式

缺点：CPU使用率高

## 步骤：
* 1. 启动ADC
* 2. 等待EOC标志位
* 3. 读取寄存器数据

## 主要函数

```c++
// 开启ADC
HAL_StatusTypeDef       HAL_ADC_Start(ADC_HandleTypeDef *hadc);
// 关闭ADC
HAL_StatusTypeDef       HAL_ADC_Stop(ADC_HandleTypeDef *hadc);
// 等待转换完成标志位，阻塞等待
HAL_StatusTypeDef       HAL_ADC_PollForConversion(ADC_HandleTypeDef *hadc, uint32_t Timeout);
// 获取ADC采样值
uint32_t HAL_ADC_GetValue(ADC_HandleTypeDef *hadc);


```

## 单通道查询方式

Cubemx配置：

<image src="./img/单通道查询.png">

由于我们只使用了一个ADC，所以设置为独立模式。

由于只使用了ADC1的一个通道，所以关闭扫描模式和间断模式。

我们希望每次需要ADC转换时才开启ADC，做完一次转换就停止。所以连续模式设置为Disable。

规则组ADC触发方式选择为 **软件触发**

Keil中C++代码，由于单次采样模式，所以每次获取ADC值时都要重新开启一下ADC：

```c++
float get_adc_value(void)
{
	HAL_ADC_Start(&hadc1); // 开启ADC
	HAL_ADC_PollForConversion(&hadc1,50); // 等待完成转换
	
	// 进一步确认是否完成转换
    if(HAL_IS_BIT_SET(HAL_ADC_GetState(&hadc1),HAL_ADC_STATE_REG_EOC))
	{
		return HAL_ADC_GetValue(&hadc1);
	}
	else
	{
		return 0;
	}
}
```
## 多通道查询方式

**使用多通道需要要开启扫描模式**

### 典型错误
不建议使用使用查询方式进行多通道。以下是典型错误使用：

<image src="./img/查询方式多通道采集典型错误.png">

这里我们使能了多个ADC通道，但是其他的配置仍然和单通道查询时相同。

仅使用ADC2，独立模式。并使用ADC2的4个通道，设置规则组通道数为4，Rank1234分别为这4个通道。启用Scan Conversion Mode，关闭Continuous Conversion Mode，关闭Discontinuous Conversion Mode。

因为每个ADC的规则组只有一个数据寄存器`ADC_DR`。使用一个ADC模块进行查询模式多通道采集时，数据寄存器ADC_DR中的值会不断被最新的数据覆盖，导致无法拿出多通道的数据。

### 解决办法——使用间断模式

<image src="./img/间断模式.png">

开启间断模式。并且设置了 Number Of Discontinuous Conversions为1。其他配置与上面相同。

也就是说，每一个间断组中是一个通道。第一次开启ADC转换时，将第一个间断组（也就是ch1）进行转换，然后第二次开启时，进行第二个间断组（也就是ch2），依次进行完四个间断组之后，第五次使能ADC再从第一个间断组开始。

这样就避免了每次使能ADC转换时多个通道同时向`ADC_DR`中写数据导致错误。

**注意，即便是可以用间断模式解决问题，但是要尽量避免用查询方式进行多通道ADC转换**

# 三、中断方式

## 步骤：

* 1. 启动ADC、使能中断
* 2. EOC自动触发中断
* 3. 在中断回调函数中读取寄存器数据

## 主要函数

```c++ 
// 开启ADC，并使能中断
HAL_StatusTypeDef       HAL_ADC_Start_IT(ADC_HandleTypeDef *hadc);
// 关闭ADC 关闭中断
HAL_StatusTypeDef       HAL_ADC_Stop_IT(ADC_HandleTypeDef *hadc);
//中断回调函数，需要重写
__weak void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc);
```

## 中断方式 多通道采集

因为使用中断CPU不需要等待转换完成，这里可以使能连续转换模式，就不用在while(1)中多次开启ADC了。

使能扫描模式和连续模式：
<image src="./img/中断多通道.png">

使能ADC中断：

<image src="./img/ADC中断.png">

keil中首先在初始化中开启ADC和使能中断：

```c++
HAL_ADC_Start_IT(&hadc1);
```

然后重写ADC中断回调函数：
```c++
// 重写中断回调函数
float adc_value[20];
uint8_t n;
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    adc_value[n++] = HAL_ADC_GetValue(hadc) * FACTOR_ADC; // 获取ADC数据并且存到数组中去
}

```
最终，数组中数据存放是按照Cubemx配置中指定的Rank顺序**依次存放**。


# 四、DMA模式

* 1. 启动ADC
* 2. 配置DMA缓存区
* 3. 读取缓存区数据

## 4.1 主要函数

```c++
HAL_StatusTypeDef       HAL_ADC_Start_DMA(ADC_HandleTypeDef *hadc, uint32_t *pData, uint32_t Length);
HAL_StatusTypeDef       HAL_ADC_Stop_DMA(ADC_HandleTypeDef *hadc);
```
## 4.2 DMA 多通道采集

### 4.2.1 循环一直不停地进行DMA方式的ADC采样

<image src="./img/DMA模式配置.png">

使能`DMA Continuous Requests`(DMA连续请求模式)，使能扫描模式和连续模式。

添加DMA通道：

<image src="./img/DMA通道配置.png">

* DMA请求就是使用的ADC外设
* 通道可以使用默认的，也可以自己选一个
* 方向是从外设到内存
* 优先级根据情况
* 使用循环模式，数组满了之后从头开始覆盖
* ADC外设数据寄存器只有一个，只有一个地址，所以不自增
* 内存地址自增
* ADC数据寄存器是16位，半个World长度，选择Half World，每次搬运16位。

keil中，首先在主循环之前，初始化中开启DMA

```c++
uint16_t adc_value[100]; //全局变量，用于存放DMA搬运来的数据
//开启DMA，并且指定内存中存放数据起始地址和长度。
HAL_ADC_Start_DMA(&hadc1,(uint32_t *)adc_value, 40);
```
因为这个函数第二个参数要求的是32位的指针，所以这里做一个强制类型转换

* 由于DMA采用了连续传输的模式，ADC采集到的数据会不断传到到存储器中（此处即为数组ADC_Value）。ADC采集的数据从ADC_Value[0]一直存储到ADC_Value[99]，然后采集到的数据又重新存储到ADC_Value[0]，一直到ADC_Value[99]。所以ADC_Value数组里面的数据会不断被刷新。这个过程中是通过DMA控制的，不需要CPU参与。我们只需读取ADC_Value里面的数据即可得到ADC采集到的数据。其中ADC_Value[0]为通道0(PA0)采集的数据，ADC_Value[1]为通道1(PA1)采集的数据，ADC_Value[2]为通道0采集的数据，如此类推。数组偶数下标的数据为通道0采集数据，数组奇数下标的数据为通道1采集数据 , 在get_adc_value中 ，我们将通道0跟通道1采集到的数据，除以50，达到滤波的效果。

```c++
uint32_t get_adc_value()
{
	 for(i =0,ad1= 0,ad2= 0; i < 100 ; ) 
		{
		    ad1 += adc_value[i++];
			ad2 += adc_value[i++];
		}
		ad1= adc_value[0];
		ad2 = adc_value[1];
		lcd_show_num(1,1,ad2,6,16,BLACK);
		lcd_show_num(1,20,ad1,6,16,BLACK);
		ad1 /= 50;
		ad2 /= 50;
}
```
### 4.2.2 DMA 单次采集

一种情况是可以关闭连续转换模式，这样每次调用	`HAL_ADC_Start_DMA()`之后，只会进行一轮ADC转换，完成规则组中所有的ADC采样后，ADC就关闭。等到下一次需要进行ADC采样时，再开启`HAL_ADC_Start_DMA()`

另一种是可以关闭`DMA Continuous Requests`，每次开启`HAL_ADC_Start_DMA()`之后，当数组ADC_Value[100]采集满时，ADC和DMA就停止了。下次需要的时候再开启`HAL_ADC_Start_DMA()`





# 多重模式


# 校准

在Cubemx生成的初始化函数`MX_ADCx_Init();`之后进行一次校准：

`HAL_ADCEx_Calibration_Start(&hadc1 );//adc校准`



 CubeMX所生成的代码默认将DMA初始化放到ADC初始化后面，必须要将DMA初始化放到ADC初始化之前ADC才能够正常工作，否则ADC无法工作。
