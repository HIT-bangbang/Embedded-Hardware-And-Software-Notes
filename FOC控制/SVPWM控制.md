# 一、绪论

在《从SPWM到SVPWM》中，讨论了SPWM和SVPWM的推导原理，并推导了这两种控制方式的各个指标

现在我们聚焦SVPWM，推导SVPWM的矢量合成，扇区判断，以及实际的STM32的PWM生成，电流采样等流程

# 二、SVPWM矢量合成

## 2.1 回顾，为什么要电流采样和逆park变换

在上《从SPWM到SVPWM》中，我们以第一扇区为例，直接用两个基本电压矢量 $V4,V6$ 合成了所需的合成矢量 $V_\delta$

在只有电压控制，而没有电流采样的情况下，这是可以的，我们让 $u_q = V_\delta$，并且让 $u_d = 0$，这样直接用SVPWM的理论去合成 $V_\delta$

但是，正如我们在上一节最后提到的问题，电机是感性原件，电流始终是滞后电压的变化的，没有电流采样的话，很难保证 $i_d = 0$

所以，当我们的硬件有了电流采样之后，就不能这么简单的直接合成一个 $u_q$ 就行了。我们要在控制 $i_q$ 达到我们的要求的前提下，同时控制尽量让$i_d = 0$。这样的话，我们需要输出的 $u_d$ 可就不一定等于零了，这样$V_\delta$ 也就不一定与 $u_q$重合了 。

所以，我们需要两个PID，分别控制根据采样计算得到的 $i_d$ 和 $i_q$ 和参考值 $i_{d}^{ref} = 0$ 和 $i_{q}^{ref}$ 得到实际应该输出的 $u_d, u_q$ 。其实现在的 $u_d, u_q$ 合成之后就已经是我们想要输出的电压矢量 $V_\delta$ 了，按照《从SPWM到SVPWM》中的方法，可以直接求当前扇区的两个基本矢量的作用时间。但是为了计算的方便，我们还是像SPWM中介绍的，再来一步逆park变换，变换为静止系的 $u_\alpha, u_\beta$

<image src="电流力矩控制.jpg">

## 2.2 扇区判断

首先将合成电压矢量 $u_s$ 分解到$\alpha \beta$ 坐标系中，有：

$$
\begin{aligned}
& u_\alpha=u_s \cdot \cos \theta \\
& u_\beta=u_s \cdot \sin \theta \\
& \tan \theta=\frac{\sin \theta}{\cos \theta}=\frac{u_\beta}{u_\alpha} \\
& \theta=\arctan \frac{u_\beta}{u_\alpha}
\end{aligned}
$$


第一扇区： $0  \degree <\theta  <60  \degree$ 即 $0 \degree  < \arctan \frac{u_\beta}{u_\alpha}<60 \degree$

第二扇区： $60 \degree < \theta < 120\degree$ 即 $60^{\circ} < \arctan \frac{u_\beta}{u_\alpha}<120\degree$

第三扇区： $120\degree < \theta < 180\degree$ 即 $120\degree < \arctan \frac{u_\beta}{u_\alpha}<180\degree$

带入 $\theta$ 的计算可以得到：

第一扇区： $0<\frac{u \beta}{u_\alpha}<\sqrt{3}$ 且 $u_\alpha>0, u_\beta>0$

第二扇区： $\frac{u_\beta}{u_\alpha}<-\sqrt{3}, \frac{u_\beta}{u_a}>\sqrt{3}$ 且 $u_\beta>0$

第三扇区： $-\sqrt{3}<\frac{u \beta}{u_\alpha}<0$ 且 $u_\alpha<0, u_\beta>0$

**注意：** 正切函数的性质，需要判断$u_\alpha$和$u_\beta$的正负

也就是说，在 $u_\beta>0$ 的情况下：

* 第一扇区： $\frac{\sqrt{3}}{2}{u_\alpha} - \frac{1}{2}u_\beta > 0$

* 第三扇区： $- \frac{\sqrt{3}{u_\alpha}}{2} - \frac{1}{2}u_\beta < 0$

* 第二扇区： 前面两种都不符合

这里多乘了个$\frac{1}{2}$，可以简化运算，后面再说


为了方便总结规律，我们先定义三个中间变量：

$$
\begin{equation}
\begin{cases}
    U_{ref1} = u_\beta \\
    U_{ref2} = \frac{\sqrt{3}}{2}{u_\alpha} - \frac{1}{2}u_\beta\\
    U_{ref3} =  - \frac{\sqrt{3}{u_\alpha}}{2} - \frac{1}{2}u_\beta\\
\end{cases}
\end{equation}
$$

然后，定义再定义三个变量$ABC$，令

* 若 $U_{ref1} > 0$，则$A = 1$，否则$A = 0$
* 若 $U_{ref2} > 0$，则$B = 1$，否则$B = 0$
* 若 $U_{ref3} > 0$，则$C = 1$，否则$C = 0$

按照二进制，把ABC排列到由低到高的数位，形成一个二进制数“$N = CBA$”

例如 $A = 1, B = 0, C = 1$， 则 $N = 101 = 5$

可以整理得到下表：

|N|3|1|5|4|6|2|
|----|----|----|----|----|----|----|
|扇区   |1|2|3|4|5|6|


## 2.3 矢量合成

* 现在我们的SVPWM模块要解决的问题是：

    通过 $u_\alpha, u_\beta$ 计算当前扇区的两个基本矢量的作用时间，从而得出三相的PWM值


<image src="V向量合成.png">

如图所示，首先，有

$$
\begin{cases}
    u_\alpha = V_{4} \cdot \frac{T_4}{T_s} + V_{6} \cdot \frac{T_6}{T_s} \cdot \cos\frac{\pi}{3}
    \\
    \\
    u_\beta =  V_{6} \cdot \frac{T_6}{T_s} \cdot \sin\frac{\pi}{3} 
    \\
    \\
    V_1 = V_6 = \frac{2}{3}V_{dc}
\end{cases}
$$

**注意：** 这里有一点不同，我们令$V_1$到$V_6$的大小为$\frac{2}{3}V_{dc}$，而不是之前的$V_1 = V_6 = V_{dc}$ 这会带来一些计算上的方便。

化简得到：

$$
\begin{cases}
    u_\alpha = V_{dc} \cdot \frac{2T_4 +T_6}{3T_s} 
    \\
    \\
    u_\beta =  V_{dc} \cdot \frac{\sqrt{3} T_6}{3T_s}
\end{cases}
$$

联立方程，解出：

$$
\begin{cases}
    T_4 = \frac{\sqrt{3} T_s}{V_{dc} } \cdot (\frac{\sqrt{3}}{2}u_\alpha - \frac{1}{2}u_\beta )
    \\
    \\
    T_6 =  \frac{\sqrt{3} T_s}{V_{dc}} \cdot u_\beta
\end{cases}
$$

这样似乎比《从SPWM到SVPWM》中用正弦定理算还简单一点~~

为了方便表示，我们定义三个中间变量：

$$
\begin{equation}
\begin{cases}
    X =  \frac{\sqrt{3} T_s}{V_{dc}} \cdot u_\beta
    \\
    \\
    Y = \frac{\sqrt{3} T_s}{V_{dc} } \cdot (\frac{\sqrt{3}}{2}u_\alpha + \frac{1}{2}u_\beta )
    \\
    \\
    Z = \frac{\sqrt{3} T_s}{V_{dc} } \cdot (-\frac{\sqrt{3}}{2}u_\alpha + \frac{1}{2}u_\beta )
\end{cases}
\end{equation}
$$

这样，我们就可以总结一张表：

<image src="N和XYZ对应表.jpg">

结合扇区判断章节的表格，这就完成了所有的推导。

## 2.3 变量共用与计算简化

在2.1章节扇区判断的公式 $(1)$ 和2.2章节矢量合成的公式$(2)$中，对比这两个公式不难发现，有两部分其实是共用的：

$$
\begin{cases}
    M = \frac{\sqrt{3}}{2}u_\alpha + \frac{1}{2}u_\beta
    \\
    N = -\frac{\sqrt{3}}{2}u_\alpha + \frac{1}{2}u_\beta
\end{cases}
$$

这在代码里面只需要计算一次

2.1章节中乘的那个$\frac{1}{2}$就是为了共用这个代码

# 2.4 电桥状态切换时刻计算

以七段式为例：

<image src="七段式完整波形1.jpg">

设$a,b,c$三相第一次由0变为1的时刻为 $T_a$， $T_b$， $T_c$

那么第一扇区的切换时间为：

$$
\begin{cases}
    T_a = \frac{T_0}{4} =  \frac{T_s - T_4 - T_6}{4}
    \\
    \\
    T_b = T_a + \frac{T_4}{2}
    \\
    \\
    T_c = T_b + \frac{T_6}{2}
\end{cases}
$$

在第二扇区，对比第一扇区的图不难发现，第二扇区的abc相切换时间$T_a^\prime T_b^\prime T_c^\prime$的计算方法分别和第一扇区的$T_b$， $T_a$， $T_c$是相同的，可以获得这样的对应：

$$
\begin{cases}
T_a^\prime = T_b \\
T_b^\prime = T_a \\
T_c^\prime = T_c    
\end{cases}
$$

依次类推，我们可以得到六个扇区的时间切换对应表：

<image src="各区切换时间汇总表.jpg">

至此，所有理论的推导已经完成。还有一些关于具体的STM32高级定时器的使用和电流采样时机，在下一章记录。








