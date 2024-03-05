# 一、端电压、相电压、线电压

<image src="电机原理图.png">

上图中，端电压为$U_{AO}$、$U_{BO}$、$U_{CO}$，也就是电机的引线ABC与驱动电路的GND之间的电压

线电压为$U_{AB}$、$U_{BC}$、$U_{AC}$，也就是电机三条线两两之间的电压。

相电压为$U_{AN}$、$U_{BN}$、$U_{CN}$，也就是ABC与中性点N之间的电压，即每一相绕组线圈两端的电压。

# 二、电压利用率

## 2.1 电压利用率定义
* **注意**：全桥电路控制的是**端电压**

定义电压利用率为：

$$\text{电压利用率} = \frac{\text{线电压幅值}}{\text{直流母线电压}}$$

直流母线电压就是给驱动板MOS供电的直流电电压VCC

例如：

我们想要设置A点的端电压值$U_{AO}$时，M1的占空比直接决定A点的端电压，而不管其他两相的MOS管是怎样的。同理可以设置B和C的端电压

## 2.2 SPWM的电压利用率

当控制M1的占空比为100%时：  端电压 $U_{AO}=V_{dc}$ 

当控制M1的占空比为50%时：   端电压 $U_{AO}=(1/2)V_{dc}$

当控制M1的占空比为0%时：    端电压 $U_{AO}=0$

基于以上分析，可以得到三相端电压的表达式：

$U_{AO}= \frac{1}{2} V_{dc} \cos(\theta) + \frac{1}{2} V_{dc}$

$U_{BO}= \frac{1}{2} V_{dc} \cos(\theta - 120\degree) + \frac{1}{2} V_{dc}$

$U_{CO}= \frac{1}{2} V_{dc} \cos(\theta + 120\degree) + \frac{1}{2} V_{dc}$

三相端电压波形图如下所示：
<image src="SPWM端电压波形.png">

根据端电压与相电压的关系，可以得到下列公式：

$$U_{AN} = U_{AO} - U_{NO}$$
$$U_{BN} = U_{BO} - U_{NO}$$
$$U_{CN} = U_{CO} - U_{CO}$$

等式左右分别相加可以得到：$U_{AN}+U_{BN}+U_{CN}=U_{AO}+U_{AO}+U_{AO}-3U_{NO}$

同一时刻三相相电压相加的和为0（基尔霍夫定律，电流流入=流出），因此：

$$U_{AN} + U_{BN} + U_{CN} = U_{AO} + U_{BO} + U_{CO} - 3U_{NO}=0$$

可以得到：

$$U_{NO} = \frac{U_{AO}+U_{BO}+U_{CO}}{3} = \frac{1}{2}V_{dc}$$

进而得到相电压公式：

$$U_{AN} = U_{AO} - U_{NO} = \frac{1}{2} V_{dc} \cos(\theta)$$
$$U_{BN} = U_{BO} - U_{NO} = \frac{1}{2} V_{dc} \cos(\theta - 120\degree)$$
$$U_{CN} = U_{CO} - U_{CO} = \frac{1}{2} V_{dc} \cos(\theta + 120\degree)$$

基于上式可知，相电压的幅值为 $\frac{1}{2}V_{dc}$。再根据相电压与线电压的关系，可以得到线电压的幅值为 $\frac{\sqrt{3}}{2}V_{dc}$:

<image src="相电压与线电压矢量关系图.jpg">

电压利用率为：

$$ \text{电压利用率} = \frac{\text{线电压幅值}}{\text{直流母线电压}}  = \frac{\frac{\sqrt{3}}{2}V_{dc}}{V_{dc}}$$


## 2.3、SVPWM电压利用率

<image src="svpwm扇区图.png">

SVPWM的算法中，线性调制的条件就是电压矢量不超过六边形的边界，而当目标参考电压矢量的末端运动轨迹是上图正六边形的内切圆时，达到SVPWM的线性调制上限。

也就是线性调制区和过调制区的临界条件就是电压矢量的最大值为内切圆的半径$(\sqrt{3}/3)V_{dc}$，因此相电压基波幅值为$(\sqrt{3}/3)V_{dc}$。

根据相电压与线电压的关系，可以得到线电压的幅值为$\left( \sqrt{3} \right)(\sqrt{3}/3)V_{dc}=V_{dc}$ 。

$$电压利用率=\frac{线电压幅值}{直流母线电压} = \frac{V_{dc}}{V_{dc}}=1$$ 

所以算下来SVPWM比SPWM的电压利用率要高

# 三、SPWM

## 3.1 由发电机到电动机
发电机发出的三相电有这样的性质：

* 假设中性点**电势**是个定值，则三个接出点对于中性点的电压，也就是相电压，是三个相位间隔120度分布的正弦波
* 中性点的电势可以随意假设，如果假设为GND，那么相电压其实也就和各自的端电压相等了。


由发电机发出的三相电逆推，输入三相相位呈120度分布的三相正弦**相电压**也能生成旋转的磁场，但是我们没办法直接控制相电压，因为中性点在电机内部，没接出来。只能直接控制端电压，从而控制线电压。

如果假设中性点电压为0V，那么要生成的相电压就在0V上下正弦波动。这样就会产生在0V上下正弦波动的端电压，显然我们搭建的MOS管电路并不能产生负电压。所以我们做一件事情，就是将端电压波形提高$\frac{1}{2}V_{dc}$，让它在$\frac{1}{2}V_{dc}$上下正弦波动，幅值最大可以取得$\frac{1}{2}V_{dc}$。

给电机通三相相差120度的**端电压**带可以推导出以下结论：

* 相电压，线电压，相电流，线电流都是相位差120度的三个正弦信号
* 中性点$V_{NO}$电压不变，为$0.5V_{dc}$

## 3.2 Clark变换

设三相绕组的相电流矢量分别为$I_a$ $I_b$ $I_c$，这三个电流矢量方向的三个单位向量构成了一组单位基向量。

合成的电流矢量表示为$I_s$，它代表了当前的状态。$I_s$的幅值为$\frac{3}{2} I_m$

$I_s$可以用$i_a$ $i_b$ $i_c$三个单位基向量的线性组合表示。

$I_s$也可以用一组单位正交基$i_\alpha$和$i_\beta$的线性组合表示。（二维空间两个基就可以表示空间中的任意矢量了）

所以可以将$I_a$ $I_b$ $I_c$转换到直角坐标系 $\alpha \beta$ 系，即用互相垂直的$I_\alpha$和$I_\beta$表示。

不管是$I_a$ $I_b$ $I_c$还是$I_\alpha$和$I_\beta$都是表示了当前的电流的状态$I_s$，他们之间的转换是可逆的并且是唯一的。

为了更加方便，规定$I_a$和$I_\alpha$对齐，也就是将$I_\alpha放在$a绕组方向上。并且$I_a$正方向为产生磁场N级的方向

$$
\begin{cases}
I_{\alpha}=I_a+cos(\frac{2\pi}3)I_b+cos(\frac{2\pi}3)I_c \\
I_{\beta}=sin(\frac{2\pi}3)I_b-sin(\frac{2\pi}3)I_c
\end{cases}
$$

写成矩阵形式如下：

$$\begin{bmatrix}I_{\alpha}\\I_{\beta} \end{bmatrix}=\begin{bmatrix}1 & -\frac12 & -\frac12 \\ 0 & \frac{\sqrt3}2 & -\frac{\sqrt3}2 \end{bmatrix} \begin{bmatrix}I_a\\I_b\\I_c \end{bmatrix}$$

$I_\alpha$和$I_\alpha$的幅值为$\frac{3}{2} I_m$

<image src="clark变换波形.jpg">

如上图所示，可见变换之后的$I_\alpha$和$I_\beta$依然是正弦波，我们需要控制的变量减少了，由三个变成了两个。

当然，我们直接控制的是电压，将相电流Clark变换改为相电压（除以绕组电阻，实际上不精确，还有感抗和容抗）可以得到：

$$\begin{bmatrix}U_{\alpha}\\U_{\beta} \end{bmatrix}=\begin{bmatrix}1 & -\frac12 & -\frac12 \\ 0 & \frac{\sqrt3}2 & -\frac{\sqrt3}2 \end{bmatrix} \begin{bmatrix}U_a\\U_b\\U_c \end{bmatrix}$$

这里需要注意的是，电压Clark变换里面用到的是相电压$U_a = U_{AN}$、$U_b = U_{BN}$、$U_c = U_{CN}$，而实际上MOS管逆变电路控制的是端电压$U_{AO}$、$U_{BO}$、$U_{CO}$

Clark变换中所有的坐标系都是静止的，固连在定子绕组上的。

## 3.3 Park变换
控制电机时我们并不能直接测量得到 $I_\alpha$ $I_\beta$，我们已知的是转角。

所以最好是用转角和一个定值得到合向量$I_s$（其实是极坐标表示），然后转换为$I_\alpha$ $I_\beta$，最后转换为三相电流。

再设置一个坐标系，称为QD系。QD系固连在转子上，并且假定D轴与转子的S级方向重合，随转子一起同步转动。

先让$QD$系一开始与${\alpha}{\beta}$系坐标轴重合，步骤如下：

让$I_{\alpha}$取到最大，这样的时候转子N级沿着$I_{\alpha}$轴方向，这时转子的S级方向与定子N级吸合。此时D轴与$I_{\alpha}$轴重合。后面在说零电位校准的时候其实也是在做这个步骤。

电机转动起来的某一时刻，$QD$系与${\alpha}{\beta}$系错开了电角度$\theta$
<image src="Park变换.png">

$$\begin{cases}I_d=I_{\alpha}cos(\theta)+I_{\beta}sin(\theta)\\I_q=-I_{\alpha}sin(\theta)+I_{\beta}cos(\theta)\end{cases}$$

矩阵形式：

$$\begin{bmatrix}I_d\\I_q \end{bmatrix}=\begin{bmatrix}cos{\theta} & sin{\theta} &\\ -sin{\theta} & cos{\theta}\end{bmatrix} \begin{bmatrix}I_{\alpha}\\I_{\beta} \end{bmatrix}$$

通过编码器可以获得转子的实时旋转角度，所以这个角度​是一个已知数。经过这一步的变换，我们会发现，一个匀速旋转向量在这个坐标系下变成了一个定值！（显然的嘛，因为参考系相对于向量$I_s$静止了），这个坐标系下两个控制变量都被线性化了！


## 3.4 Park变换和Clark变换是怎样让电机转起来的

转子是一块存在于定子磁场中的长条磁铁。同性相斥异性相吸，只要定子产生的磁场与转子磁场错开，让丁次磁场N级沿着Q轴方向，就可以产生最大的力矩使转子转起来了。当然不垂直也是可以的，只要叉开就可以产生力矩，只不过垂直的时候力矩是最大的（是吗？）。

合向量$I_s$其实就是定子合磁场的方向（N级方向）。让$I_s$一直保持垂直转子磁场就可以一直产生力矩。

* 我们先指定一个$I_s$的大小，也就是定子磁场的强度
* 然后根据定子的转角，确定$I_s$的方向，$I_s$应该保持指向转子坐标系的Q轴。
* 将$I_s$转换到定子坐标系中，求得$I_\alpha$ $I_\beta$ ————Park逆变换
* $I_\alpha$ $I_\beta$ 转换到$i_a$ $i_b$ $i_c$坐标系中，得到三相电流$I_a$ $I_b$ $I_c$ ————Clark逆变换
* 当然，实际控制的是电压，也有上面的步骤，不过最后有一个相电压到端电压的换算，加上中性点电压即可（见章节2.2）

## 3.5 FOC矢量控制的本质

**产生转矩**的根本是：让定子线圈产生一个垂直转子的磁场。

这个人为产生的磁场的方向就是需要生成的合电流矢量$I_s$的方向，它需要沿着Park变换中的Q轴方向才能产生最大**力矩**

clark变化其实是建立了一个复平面，也可以将电流矢量表示为复数。

Park变换和反变换其实很无聊……$QD$系换成极坐标系更方便理解（因为D轴没用……）。因为Park变换最终是将$I_\alpha$ $I_\beta$转换为了$I_s$，（$I_s$只用一个模长和角度 ${\theta}$ 就可以描述）。

* Clark变换和Park变换的过程：

    $I_a$ $I_b$ $I_c$ --> $I_\alpha$ $I_\beta$ --> $I_s$

* 逆变换：

    给定$I_s$ --> $I_\alpha$ $I_\beta$ --> $I_a$ $I_b$ $I_c$

其实整个Clark和Park变化的最终目的是将三相的电流用一个合矢量$I_s$来表示，而它用模长和转角就可以描述。

**下面绕过clark变换和park变化，直接用三相电流合成：**

$I_a$ $I_b$ $I_c$ 在空间上呈120°间隔分布，在相位上也相差120°，注意这里是时间和空间都相差120度。

给电机加上给定的三相正弦电流，设三相电流的**大小**随时间变化表达式为:

$$
\begin{cases}
    I_a=I_{m}cos(\omega t)
    \\I_b=I_{m}cos(\omega t - \frac{2\pi}{3})
    \\I_c=I_{m}cos(\omega t + \frac{2\pi}{3})
\end{cases}
$$

合成矢量为：

$$\vec{I_s} = \vec{I_a} + \vec{I_b} + \vec{I_c} $$

以$\vec{I_a}$方向为实轴建立复平面，在复平面表达为：

$$\vec{I_s} = I_{m}cos(\omega t) + I_{m}cos(\omega t - \frac{2\pi}{3})e^{j\frac{2\pi}{3}} + I_{m}cos(\omega t + \frac{2\pi}{3})e^{-j\frac{2\pi}{3}} $$

根据欧拉公式：

$$ e^{jx} = \cos x + j \sin x$$

$$
\left\{\begin{array}{l}
\vec{I}_s=I_m \cos (\omega t)+I_m \cos \left(\omega t-\frac{2 \pi}{3}\right)\left(\cos \frac{2 \pi}{3}+j \sin \left(\frac{2 \pi}{3}\right)\right)+I_m \cos \left(\omega t+\frac{2 \pi}{3}\right)\left(\cos \frac{2 \pi}{3}-j \sin \frac{2 \pi}{3}\right) \\
=I_m \cos (\omega t)+I_m\left(\cos (\omega t) \cos \frac{2 \pi}{3}+\sin (\omega t) \sin \frac{2 \pi}{3}\right)\left(-\frac{1}{2}+j \frac{\sqrt{3}}{2}\right) + I_m\left(\cos (\omega t) \cos \frac{2 \pi}{3}-\sin (\omega t) \sin \frac{2 \pi}{3}\right)\left(-\frac{1}{2}-j \frac{\sqrt{3}}{2}\right) \\
=I_m \cos (\omega t)+I_m\left(-\frac{1}{2} \cos (\omega t)+\frac{\sqrt{3}}{2} \sin (\omega t)\right)\left(-\frac{1}{2}+j \frac{\sqrt{3}}{2}\right)+I_m\left(-\frac{1}{2} \cos (\omega t)-\frac{\sqrt{3}}{2} \sin (\omega t)\right)\left(-\frac{1}{2}-j \frac{\sqrt{3}}{2}\right) \\
=I_m \cos (\omega t)+I_m\left(\frac{1}{4} \cos (\omega t)-\frac{\sqrt{3}}{4} \sin (\omega t)-j \frac{\sqrt{3}}{4} \cos (\omega t)+j \frac{3}{4} \sin (\omega t)\right)+
I_m\left(\frac{1}{4} \cos (\omega t)+\frac{\sqrt{3}}{4} \sin (\omega t)+j \frac{\sqrt{3}}{4} \cos (\omega t)+j \frac{3}{4} \sin (\omega t)\right) \\
=I_m\left(\frac{3}{2} \cos (\omega t)+j \frac{3}{2} \sin (\omega t)\right) \\
=\frac{3}{2} I_m e^{j \omega t}
\end{array}\right\}
$$

三相合成矢量是一个角速度为$\omega t$旋转且绕中心点旋转的矢量，它的幅值是相电流幅值的$\frac{3}{2}$倍。

同理有电压合成：

<image src="三相电压合成.png">

$$
\left\{\begin{array}{l}
\vec{U}_s=U_m \cos (\omega t)+U_m \cos \left(\omega t-\frac{2 \pi}{3}\right)\left(\cos \frac{2 \pi}{3}+j \sin \left(\frac{2 \pi}{3}\right)\right)+U_m \cos \left(\omega t+\frac{2 \pi}{3}\right)\left(\cos \frac{2 \pi}{3}-j \sin \frac{2 \pi}{3}\right) \\
=U_m \cos (\omega t)+U_m\left(\cos (\omega t) \cos \frac{2 \pi}{3}+\sin (\omega t) \sin \frac{2 \pi}{3}\right)\left(-\frac{1}{2}+j \frac{\sqrt{3}}{2}\right) + U_m\left(\cos (\omega t) \cos \frac{2 \pi}{3}-\sin (\omega t) \sin \frac{2 \pi}{3}\right)\left(-\frac{1}{2}-j \frac{\sqrt{3}}{2}\right) \\
=U_m \cos (\omega t)+U_m\left(-\frac{1}{2} \cos (\omega t)+\frac{\sqrt{3}}{2} \sin (\omega t)\right)\left(-\frac{1}{2}+j \frac{\sqrt{3}}{2}\right)+U_m\left(-\frac{1}{2} \cos (\omega t)-\frac{\sqrt{3}}{2} \sin (\omega t)\right)\left(-\frac{1}{2}-j \frac{\sqrt{3}}{2}\right) \\
=U_m \cos (\omega t)+U_m\left(\frac{1}{4} \cos (\omega t)-\frac{\sqrt{3}}{4} \sin (\omega t)-j \frac{\sqrt{3}}{4} \cos (\omega t)+j \frac{3}{4} \sin (\omega t)\right)+
U_m\left(\frac{1}{4} \cos (\omega t)+\frac{\sqrt{3}}{4} \sin (\omega t)+j \frac{\sqrt{3}}{4} \cos (\omega t)+j \frac{3}{4} \sin (\omega t)\right) \\
=U_m\left(\frac{3}{2} \cos (\omega t)+j \frac{3}{2} \sin (\omega t)\right) \\
=\frac{3}{2} U_m e^{j \omega t}
\end{array}\right\}
$$

<image src="三相电压合成动图.gif">

## 3.6 关于步进电机和BLDC电机FOC控制的区别

如果直接给三相绕组通这样的三相交流电的话，定子产生的合磁场就是沿着$I_m$或者$U_m$的方向，箭头就是N级，以角速度$\omega$不断转动。转子也会吸合在这个方向上一起转动，这其实就是步进电机的原理，是天生的位置控制。但是！对于BLDC电机，转子磁场和与定子磁场吸合的时候是力矩为0的时候，两个磁铁只有错开才会产生力矩，错开90°的时候才是力矩最大的时候。对于BLDC电机，我们要始终让定子合磁场的方向垂直转子磁场，以产生最大的转矩。所以BLDC电机的控制本质是力矩控制。

为什么不能像步进电机那样，利用转子磁极吸合定子磁场去做位置控制呢？其实是可以的，FOC开环驱动其实就是，但是发热特别严重。两个磁铁只有错开才会产生力矩，步进电机因为其独特的结构设计，稍微错开一丁点就会有很大的恢复力矩使得转子回来（当然还是会有一定的角度差来产生平衡外力的力矩），步进定子结构上先天就是个电磁铁，即便是到达了指定的位置，依然有电流。但是BLDC电机的内阻普遍较小，用开环控制时，电流特别大太热严重。而且由于结构设计的问题，极对数比步进电机的等效极对数少得多得多，导致定子转子磁场得岔开很大的角度才能产生足够的恢复力矩，根本达不到位置控制的要求。

那么步进电机可以不可以用FOC控制呢？可以，但是很难。步进电机因为其独特的结构设计，稍微错开一丁点就会有很大的恢复力矩使得转子回来。这个结构指的是特别多的齿槽，50个齿使得等效的极对数50为，机械角度换算电角度的时候乘以50就很大了，略微的机械角度偏差就会导致电角度不对，而且对于驱动电路的控制精度要求也很高。

## 3.7 等幅值Clark变换

下面推导等幅值Clark变换，主要是解释为什么等幅值变换需要乘$\frac{2}{3}$

上面我们知道了，因为我们让$I_d = 0$（$U_d = 0$），那么$I_q$（$U_q$）其实就是合成矢量$I_s$（$U_s$），以下推导并不严格区分这两个量了。

回顾Clark变换：

$$\begin{bmatrix}I_{\alpha}\\I_{\beta} \end{bmatrix}=\begin{bmatrix}1 & -\frac12 & -\frac12 \\ 0 & \frac{\sqrt3}2 & -\frac{\sqrt3}2 \end{bmatrix} \begin{bmatrix}I_a\\I_b\\I_c \end{bmatrix}$$

注意到：$I_\alpha$和$I_\beta$的幅值均为$\frac{3}{2} I_m$，而后面Park变换的$I_q$的幅值也是$\frac{3}{2} I_m$。

这牵扯到另一个问题，前面讨论的都是先假定给电机加上一个三相电流，然后推导$I_q$与合成磁场的性质。而现在我们要做的是，设定一个$I_q$（$U_q$），通过Park/Clark逆变换反推三相的电流（电压）。设定的$I_q$（$U_q$）大小决定了幅值$U_m$的大小。但是$U_q$的肯定不能设置的无限大。因为供电电压$V_{dc}$是一定的，而且我们首先假设了生成的相电流波形是相位差120度的正弦波。

所以$U_q$的范围是怎样呢？
我们先确定$U_m$是多少？
显然，在相电压达到幅值时，端电压也达到幅值，端电压最大为$V_{dc}$，对应的相电压为$0.5V_{dc}$。


由**正向推导**得到的$U_q$的大小和$U_\alpha$的幅值相同，均为$\frac{3}{2} U_m$。这里注意$U_m$是相电压的幅值，当取得最大值时$U^*_m = V_{dc}/2$。所以，设定$U_q$时，最高设置为$\frac{2}{3} U^*_m = \frac{3}{4}V_{dc}$ 。

但是这有些麻烦，不如在Clark变换时，等式右边乘一个$\frac{2}{3}$，这样$U_q$的大小、$U_\alpha$的幅值、相电压的幅值$U_m$就相等了。此时$U_q$的取值范围为$[-\frac{1}{2}V_{dc},\frac{1}{2}V_{dc}]$

这里有一个问题，只在等式一侧乘一个数字怎么看都让人觉得不放心，等式还成立吗？

这里先明确一个概念，不管是Clark变换还是Park变换，只是将三个矢量换了个表达方式。我们不停地换坐标系来简化表示这三个矢量，现在乘2/3只是增加了一个缩放。等于号"="左侧是我们人为定义的量，这是个定义，就像数学上定义中间变量$a=2b+1$一样，本身没有任何意义，但是就是能简化运算。这里的$U_\alpha$也一样它只要可以方便表达物理世界就可以了。我们也只要记住它和三相电压中间怎么换算就可以了，这样Clark逆变换和反求的时候，又除以了$\frac{2}{3}$，还是原来的。

除了可以让$U_q$的大小、$U_\alpha$的幅值、相电压的幅值$U_m$相等以外，这个$\frac{2}{3}$还可以简化表达：

$$\begin{bmatrix}I_{\alpha}\\I_{\beta} \end{bmatrix} =\frac{2}{3} \begin{bmatrix}1 & -\frac12 & -\frac12 \\ 0 & \frac{\sqrt3}2 & -\frac{\sqrt3}2 \end{bmatrix} \begin{bmatrix}I_a\\I_b\\I_c \end{bmatrix}$$

化简（可以把$I_a$$I_b$$I_c$的表达式带进去硬算，也可以用基尔霍夫定律）得到：

$$
\begin{cases}
I_{\alpha}=I_a\\
I_{\beta}=\frac{1}{\sqrt{3}}I_b-\frac{2}{\sqrt{3}}I_c
\end{cases}
$$

同理有电压关系：
$$
\begin{cases}
U_{\alpha}=U_a\\
U_{\beta}=\frac{1}{\sqrt{3}}U_b-\frac{2}{\sqrt{3}}U_c
\end{cases}
$$

再次强调，这里的$U_a$是相电压。

电流环中，采样的电流就是相电流$I_a$$I_b$$I_c$，这样$I_{\alpha}=I_a$可以带来很多计算上的便利。设定的$I_q$理论上也就等于$I_{a}$的幅值（当然也等于$I_{b}$$I_{c}$幅值），根据这一关系也很方便进行电流的软件限幅。

**拓展：**
* 等幅值变换那个2/3其实在物理上也是有意义的，除了纯数学的方法以外还有物理上的推导方式，它使用垂直的两相绕组等效替代了原本的三相绕组。两相绕组上通$I_{\alpha}$$I_{\beta}$，同样可以让转子转起来。但是$I_{\alpha}$$I_{\beta}$的幅值需要比$I_a$$I_b$$I_c$的幅值大，为$\frac{3}{2}$倍。为了使电流幅值相同，就让等效替代的两相绕组的匝数是原来的$\frac{3}{2}$倍，这样产生相同的磁力时，电流就可以小一点，和原来相同了。


# 四、为什么SPWM都是正弦量

回到问题本身来看，我们要控制电机，控制的实质是什么呢？实质是我们要生成与转子磁铁垂直的合磁场，那也就是需要一个与这个磁场同向的合电流矢量$I_\delta$，也就是这个方向的合电压矢量$V_\delta$。

(这里为了与SPWM区分，换了个符号，其实就是$I_s$和$U_s$)

前面的SPWM中，我们首先根据发电机的原理倒推，先假设了输入时间和空间上呈120°间隔分布的端电压就可以产生匀速的旋转的合磁场，带动转子运动。然后根据正弦$I_a$$I_b$$I_c$推出了$I_m$的表达式。然后在实际控制时，设定$I_m = I_q$，根据前面表达式的逆形式反求回三相的电流表达式（实际控制的是电压，和这个过程相同）

也就是说，我们先把时间和空间上呈120°间隔分布三个相电压画出来如下图所示：

<image src="三相电压.png">

SPWM做的事情就是对于转子状态，然后以某种规则从x轴找一个点，这个点对应的$U_a$$U_b$$U_c$值就是我们要在电机绕组上形成的相电压，然后就有对应的端电压。是先设定了三相波形，然后用的时候从里面找需要的状态。

**相电压**在空间上呈120°间隔分布是肯定的，因为三相电机的绕组就是这么设计的。但是我们想这么一件事情，端电压在时间（相位）上一定要呈120°间隔分布嘛？SPWM事先规定了端电压的波形必须是图中相位间隔120°的正弦波，然后推导控制方式，因为这是从发电机的推出的电动机。但是端电压一定要是正弦波吗？

我们只要在满足电学特性的前提下，能生成与转子磁铁垂直的合磁场（合电流/合电压）就可以了。如果要保持转矩平稳，那也只需要让合成相电压矢量幅值不变就行，似乎与端电压形状并没有直接的联系……



端电压除了要满足端电压范围$[0,V_{dc}]$以外，需要满足的要求只有：相电压合成电压矢量是个定值，不管角度是怎样。这样才能保证磁场不变，力矩不变，旋转平稳
$$|\vec{U_s}| = |\vec{U_a} + \vec{U_b} + \vec{U_c}| = C$$

**SPWM**直接假设了$U_{AN}$$U_{BN}$$U_{CN}$在相位上呈120°间隔分布，
$$
\begin{cases}
    U_{AN}=U_{m}cos(\omega t)+\frac{1}{2}V_{dc}\\
    U_{BN}=U_{m}cos(\omega t - \frac{2\pi}{3})+\frac{1}{2}V_{dc}\\
    U_{CN}=U_{m}cos(\omega t + \frac{2\pi}{3})+\frac{1}{2}V_{dc}
\end{cases}
$$

相加，正弦分量恰好可以相消，结果为定值：

$$U_{AO} + U_{BO} + U_{CO} = \frac{3}{2}V_{dc}$$

再由基尔霍夫电压电流定律得到相电压之代数和为0，列出关系：
$$U_{AN} + U_{BN} + U_{CN} = U_{AO} + U_{BO} + U_{CO} - 3U_{NO}=0$$

求得中性点的电压$U_{NO} = \frac{1}{2}V_{dc}$也恰好是定值。

这样的话，相电压也能求了，例如a相：

$Ua = U_{AN} = U_{A0} - U_{NO} = U_{m}cos(\omega t)$

**SPWM的一切都来源于一个设定，输入三个间隔120相位分布的端电压**，基于这个设定，直接保证了合成电压矢量模长为定值，中性点电压不变，所有量都是正弦量这一系列的优良性质。

这个设定确实很好，但是它是最优的吗？显然不是，注意到SPWM的线电压最大值，也就是上面三相线电压图像中任意两个正弦量的差值，最大似乎也没有覆盖整个$[0,V_{dc}]$的范围。

我们先抛开$U_{AN}$$U_{BN}$$U_{CN}$在时间上呈120°间隔分布的这个设定，直接从MOS管的开关推导合电压$V_\delta$的合成过程。

# 五、SVPWM

## 5.1 SVPWM推导

回到原来的这个驱动电路图：

<image src="电机原理图.png">

一共有三相桥臂，6个MOS管。每一个桥臂在任意时刻只能有一侧开启，一侧关闭，不会全开，否则就短路了；也不会全关，因为让引脚悬空也没什么意义。

定义二值函数
$$
S_x = 
\begin{cases}
1 \ \ \text{上管导通下管断开} \\
0 \ \ \text{下管导通上管断开}
\end{cases}
$$

其中 $x = a , b, c$ 表示 $a,b,c$ 三相

那么 $S_a$$S_b$$S_c$ 一共有8种组合方式，我们用 $V_i$ 表示这8种状态下三相的相电压矢量的合矢量，其中下标 $i$ 就是将 $(S_a,S_a,S_a)$ 从二进制写成十进制。其对应关系见下图：

<image src="基础向量表.jpg">


根据矢量的合成，8个$V_i$在空间中的分布如下图所示：

<image src="基础向量空间分布图.jpg">

我们建立如图中所示的坐标直角坐标系，与上面SPWM中Clark变换中定义的相同$\alpha$轴与电机$a$相绕组重合，与电流和磁场正方向同向。$\beta$轴垂直指向上方。

以$\alpha$轴为实轴，$\beta$轴为虚轴，6个非零的$V_i$将复平面分成了6个扇区。在复平面上8个$V_i$的表达式为：

$$
\begin{cases}
V_4=\left(\frac{2}{3} V_{d c}-\frac{1}{3} V_{d c} \cdot e^{j \cdot 2 \pi / 3}-\frac{1}{3} V_{d c} \cdot e^{j \cdot 4 \pi / 3}\right)=\frac{2}{3} V_{d c} \\
V_6=\left(\frac{1}{3} V_{d c}+\frac{1}{3} V_{d c} \cdot e^{j \cdot 2 \pi / 3}-\frac{2}{3} V_{d c} \cdot e^{j \cdot 4 \pi / 3}\right)=\frac{2}{3} V_{d c}\left(\frac{1}{2}+j \frac{\sqrt{3}}{2}\right)=\frac{2}{3} V_{d c} e^{j \frac{1}{3} \pi} \\
V_2=\left(-\frac{1}{3} V_{d c}+\frac{2}{3} V_{d c} \cdot e^{j \cdot 2 \pi / 3}-\frac{1}{3} V_{d c} \cdot e^{j \cdot 4 \pi / 3}\right)=\frac{2}{3} V_{d c}\left(-\frac{1}{2}+j \frac{\sqrt{3}}{2}\right)=\frac{2}{3} V_{d c} e^{j \frac{2}{3} \pi} \\
V_3=\left(-\frac{2}{3} V_{d c}+\frac{1}{3} V_{d c} \cdot e^{j \cdot 2 \pi / 3}+\frac{1}{3} V_{d c} \cdot e^{j \cdot 4 \pi / 3}\right)=-\frac{2}{3} V_{d c} \\
V_1=\left(-\frac{1}{3} V_{d c}-\frac{1}{3} V_{d c} \cdot e^{j \cdot 2 \pi / 3}+\frac{2}{3} V_{d c} \cdot e^{j \cdot 4 \pi / 3}\right)=\frac{2}{3} V_{d c} e^{j \frac{4}{3} \pi} \\
V_5=\left(\frac{1}{3} V_{d c}-\frac{2}{3} V_{d c} \cdot e^{j \cdot 2 \pi / 3}+\frac{1}{3} V_{d c} \cdot e^{j \cdot 4 \pi / 3}\right)=\frac{2}{3} V_{d c} e^{j \frac{5}{3} \pi} \\
V_0=V_7=0
\end{cases}
$$

首先要明确，$V_0$到$V_7$是电机在某一时刻可能的状态，某一时刻电机的状态只能是其中的一个。后面所谓的$V_4$$V_6$合成$V_\delta$的意思是说，极短的时间周期，电机保持在$V_4$一会儿，保持在$V_6$一会儿，由于时间周期很短，这样电机来回切换状态，就像是工作在$V_\delta$一样。这个过程非常像PWM，所以也叫SVPWM。

首先看第一个扇区，我们让电机状态在一个周期$T_s$分配给$V_4$和$V_6$，就可以等效出任意长度的$V_\delta$

<image src="V向量合成.png">
显然有

$$\vec{V_\delta} = \alpha \vec{V_4} + (1-\alpha) \vec{V_6}$$

其中$\alpha \in [0,1]$

$\left | V_4 \right| = \left | V_6 \right| = V_{dc}$

显然

$$V_{\delta \alpha} = \alpha V_{dc} + (1-\alpha)\frac{1}{2}V_{dc} = \frac{1}{2}(1+\alpha)V_{dc}$$

$$V_{\delta \beta} = (1-\alpha) \frac{\sqrt{3}}{2}V_{dc}$$

$$\theta = \arcctg\frac{V_{\delta \beta}}{V_{\delta \alpha}}$$

$$\left | V_\delta \right| = \sqrt{1-\alpha+\alpha^2} \cdot  V_{dc} $$

显然：
$\left | V_\delta \right|$的取值为$[\frac{\sqrt{3}}{2}V_{dc},V_{dc}]$

<image src="三个工作区.jpg">

对应在图上，也就是说，$\vec{V_\delta}$的边界是那个红色的正六边形。正六边形区域内是这个驱动电路能产生的合电压矢量区域，边界就是最大值的范围。

这副图其实也表达了SVPWM工作的三个区域

* 内接圆范围是线性区域，半径是$\frac{\sqrt{3}}{2}V_{dc}$，在内接圆上，我们可以设置不变的$V_\delta$大小，它在旋转的时候末端肯定一直在正六边形里面，这样的话就不用担心相电压超过物理范围
* 正六边形和内接圆之间的区域称为Ⅰ区，我们如果要使用到这个范围内的电压值的话，需要动态地随着$\theta$改变$V_\delta$大小，让它始终在正六边形之内。
* 如果我们要让电机工作在外接圆上，半径为$V_{dc}$，这只有6个顶点位置是能用的，有点类似于六步换向法？？？

前面我们考虑的是已知两个 $V_i$ 的等效大小取值范围，用 $V_i$ 合成 $V_\delta$ ，来分析 $V_\delta$ 的范围。那么下面我们来看看怎么用 $V_\delta$ 获得相邻两个 $V_i$ 的等效大小

线性区域内，如果我们要产生的电压矢量 $V_{out}$，电角度为 $\theta$ 。设时间周期为 $T_s$，$V_4$ 做工的时间为 $T_4$，$V_6$ 做工的时间为 $T_6$ ，根据正弦定理：

$$
\frac{\left| V_{out} \right|}{sin\frac{2\pi}3}=\frac{\left| \frac{T_6}T \cdot V_6 \right|}{sin\theta}=\frac{\left| \frac{T_4}T \cdot V_4 \right|}{sin(\frac {\pi}3-\theta)}
$$

其中：
$$\left|V_4\right|=\left|V_6\right|=V_{dc}$$

$$\left|V_{out}\right|=u_m$$

求得做功时间：

$T_4=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin(\frac{\pi}{3}–θ)$

$T_6=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin\left(\theta\right)$

其中，$u_m$是参考合电压矢量$V_{out}$的大小，它的最大值如前面所说的，必须在线性区之内，为$\frac{\sqrt{3}}{2}V_{dc}$。

然后把$T_4$和$T_6$都不做功的时间均分给$T_0$$T_7$

$T_0=T_7=\frac{1}{2}(T_s–T_4–T_6)$

这样应该就是算无遗漏了，应该就是能利用到所有的电压了。现在我们再来算一下相电压，线电压和端电压波形是什么样子的

## 5.2 SVPWM相电压
只考虑线性区，
以扇区1为例，$V_4$$V_6$工作的时间为：

$T_4=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin(\frac{\pi}{3}–θ)$

$T_6=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin\left(\theta\right)$

让我们来看A相绕组，在$T_s$时间周期内：$V_4$做功时，A相会以$\frac{2}{3}V_{dc}$的相电压工作$T_4$时间；在$V_6$做功时，A相会以$\frac{1}{3}V_{dc}$的相电压工作$T_6$时间，那么等效的电压也就是：

$$
\begin{aligned}
U_a &= 
\frac
{\frac{2}{3}V_{dc} \cdot T_4 + \frac{1}{3}V_{dc} \cdot T_6}
{T_s}\\
&=\frac{2\sqrt{3}}{9}u_m (2\sin(\frac{\pi}{3} - \theta) + \sin(\theta)) \\
&=\frac{2}{3}u_m \cos\theta
\end{aligned}
$$

在其他扇区，上式依然成立，证明略。

## 5.3 SVPWM端电压
先看扇区1
让我们来看A相绕组的端电压，在$T_s$时间周期内：$V_4$做功时，A相的MOS上管会打开$T_4$时间；在$V_6$做功时，A相的MOS上管打开$T_6$时间，那么等效的端电压也就是：

$$
U_{AO}= 
\frac
{T_4 + T_6 +\frac{1}{2}(1-T_4-T_6)}{T_s} V_{dc}
$$

再看扇区2，$T_2$时间内是$V_2$做功，A相MOS上管不导通，所以等效相电压为：

$$
U_{AO}= 
\frac
{T_4 +\frac{1}{2}(1-T_4-T_6)}{T_s} V_{dc}
$$

扇区3，$V_2$和$V_3$做工时，A相MOS上管都不导通：
$$
U_{AO}= 
\frac
{\frac{1}{2}(1-T_4-T_6)}{T_s} V_{dc}
$$

同理就有各个扇区的表达式。

这里可以利用一下对称性，在每个扇区中的$\theta$减去扇区开始的角度之后，就旋转到了扇区1，然后用扇区1中求出来的$T_4$和$V_6$表达式计算，我们把$\theta \in [0,2\pi]$的图像用matlab画出来（忽略了系数$\frac{1}{\sqrt{3}}u_m$）：

代码
```matlab
fplot(@(x) sin(pi/3-x)+sin(x),[0 pi/3],'b')
hold on
fplot(@(x) sin(pi/3-(x-pi/3))-sin(x-pi/3),[pi/3 2*pi/3],'b')
hold on
fplot(@(x) -sin(pi/3-(x-2*pi/3))-sin(x-2*pi/3),[2*pi/3 pi],'b')
hold on
fplot(@(x) -sin(pi/3-(x-pi))-sin(x-pi),[pi 4*pi/3],'b')
hold on
fplot(@(x) -sin(pi/3-(x-4*pi/3))+sin(x-4*pi/3),[4*pi/3 5*pi/3],'b')
hold on
fplot(@(x) +sin(pi/3-(x-5*pi/3))+sin(x-5*pi/3),[5*pi/3 2*pi],'b')
hold on
fplot(@(x) sin(pi/3-(x-2*pi))+sin(x-2*pi),[2*pi 7*pi/3],'b')
hold off
grid on
```
图像：
<image src="马鞍波.png">

这就是所谓的 ***马鞍波*** 。

## 5.4 SVPWM节点对地电压

下面推导一下$U_{NO}$的表达式：

$$U_{NO} = U_{A0}-U_{AN}$$

Matlab画图像代码（忽略提出来的系数$\frac{1}{\sqrt{3}}u_m$）
```matlab
fplot(@(x) sin(pi/3-x)+sin(x)-2*cos(x)/sqrt(3),[0 pi/3],'b')
hold on
fplot(@(x) sin(pi/3-(x-pi/3))-sin(x-pi/3)-2*cos(x)/sqrt(3),[pi/3 2*pi/3],'b')
hold on
fplot(@(x) -sin(pi/3-(x-2*pi/3))-sin(x-2*pi/3)-2*cos(x)/sqrt(3),[2*pi/3 pi],'b')
hold on
fplot(@(x) -sin(pi/3-(x-pi))-sin(x-pi)-2*cos(x)/sqrt(3),[pi 4*pi/3],'b')
hold on
fplot(@(x) -sin(pi/3-(x-4*pi/3))+sin(x-4*pi/3)-2*cos(x)/sqrt(3),[4*pi/3 5*pi/3],'b')
hold on
fplot(@(x) sin(pi/3-(x-5*pi/3))+sin(x-5*pi/3)-2*cos(x)/sqrt(3),[5*pi/3 2*pi],'b')
hold on
fplot(@(x) sin(pi/3-(x-2*pi))+sin(x-2*pi)-2*cos(x)/sqrt(3),[2*pi 7*pi/3],'b')
hold off
grid on
```

<image src="三角波.png">

这就是所谓的 ***三角波*** 。

注意，这个其实不是严格的三角波，那个线不是直的，只是近似三角波而已。

## 5.5 再次比较SPWM和SVPWM的对电压的利用率

### 5.5.1 合成电压矢量比较
* SPWM中产生的合电压矢量最大是$U_s = \frac{3}{2}U_m = \frac{3}{2} \frac{1}{2} V_{dc} = \frac{3}{4}V_{dc}$
* SVPWM中产生的合电压矢量最大是$\frac{\sqrt{3}}{2}V_{dc}$（不考虑只能在六边形顶点取得$V_\delta = V_{dc}$，工作在非线性去就不顺滑了）

### 5.5.1 线电压比较
1、SPWM中产生的线电压最大是$\frac{\sqrt{3}}{2}V_{dc}$
下面来推导这个事情：
已知SPWM的相电压是
$$
\begin{cases}
    U_a=U_{m}cos(\omega t)
    \\U_b=U_{m}cos(\omega t - \frac{2\pi}{3})
    \\U_c=U_{m}cos(\omega t + \frac{2\pi}{3})
\end{cases}
$$
其中$U_m = 0.5V_{dc}$，中性点$U_{NO} = 0$，则线电压:
$$
\begin{aligned}
U_{AB} 
&= (U_a - U_{NO})+ (U_{NO} - U_b) \\
&= U_a - U_b \\
&= U_{m}cos(\omega t) - U_{m}cos(\omega t - \frac{2\pi}{3}) \\
&= \sqrt{3} U_{m}sin(\omega t + \frac{\pi}{3})
\end{aligned}
$$
也就说两个正弦合成的正弦函数幅值为$\sqrt{3}U_{m} =\frac{\sqrt{3}}{2}V_{dc}$


2、SVPWM中产生的线电压幅值最大是$V_{dc}$，利用端电压之差（相电压之差也行）很容易求得。

### 5.5.2 SVPWM调制时的电压利用率比SPWM大15.5%

现在来算一下网上经常说的“SVPWM调制时的电压利用率比SPWM大15.5%”这件事是怎么来的：

* 算法一：从合电压矢量（合磁场）的角度来说。
$$\frac{V_\delta-U_s}{U_s} =\frac{\frac{\sqrt{3}}{2}-\frac{3}{4}}{\frac{3}{4}} = \frac{2}{\sqrt{3}}-1$$

* 算法二：从线电压的角度来说：$\frac{ V_{dc}-\frac{\sqrt{3}}{2} V_{dc} }{\frac{\sqrt{3}}{2} V_{dc}}$。


## 5.5.3、关于SVPWM和SPWM各个量的总结

对于SVPWM：

    相电压是正弦波，所以相电流一定是正弦

    线电压是正弦波？待验证

    端电压是马鞍形

    中性点电压是近似三角波

    端电压信号中，由于3相的3次谐波电流在电机中自行抵消，因此看到的相电流波形是基波分量，也就是单一的正弦波电流。

    SVPWM可以理解为注入了三次谐波的SPWM，也可以理解为注入了零序分量的SPWM。

对于SPWM：全都是正弦波。中性点电压不变。

# 六、使用SVPWM控制电机

## 6.1 做功时间分配
在扇区一中，做功时间：

$T_4=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin(\frac{\pi}{3}–\theta)$

$T_6=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin\left(\theta\right)$

剩下的时间其实让三相MOS为 $S_0$ 或者 $S_7$ 都可以。在这里我们把$T_4$和$T_6$都不做功的时间均分给$T_0$ $T_7$

$T_0=T_7=\frac{1}{2}(T_s–T_4–T_6)$

然后再来确定怎样将这几个时间分配到时间周期$T_s$中：

其实在一个时间周期内，$V_4$,$V_6$无论按照什么样的导通顺序，都可以合成所需的合成矢量，但是为了减少谐波，一般采用一下原则：

* 1、每次矢量切换只改变一个桥臂
* 2、合理利用零矢量，尽可能减少MOS管开关次数，减少开关损耗
* 3、尽可能减少谐波分量，特别是低频信号的谐波分量


### 三段法

<image src="三段法.jpg">

以扇区1为例，在安排 $V_0$,$V_4$,$V_6$ 的出现顺序时，容易想到为了保证调制算法的连续性，将零矢量$V_0$均匀的安排在起始阶段与结束阶段。即：在经过$\frac{1}{2}T_0$的$V_0$
矢量后，依次安排$T_4$时长的$V_4$矢量、$T_6$时长的$V_6$矢量，最后再续上$\frac{1}{2}T_0$时长的$V_0$矢量。如上右图所示，从上到下，分别对应a、b、c相桥臂的开关状态。

这种方法的缺点在于，某一时刻（例如图中的$T_4$到$T_6$切换时），要有两个桥臂同时动作。

### 五段法

<image src="五段法.jpg">

如上图所示，我们可以将$T_4$平均分为两份，放在$T_6$两侧，从而形成的五段式矢量序列，如上图所示。这种序列可以保证每个状态切换时，只有一相桥臂的开关动作。

其中， $T_0 = 1 - T_4 - T_6 $

### 七段法

<image src="七段法.jpg">

进一步地，在五段法的基础上，将$T_6$也劈开两等份，再中间插入$T_7$，从而形成的七段式矢量序列，如上图所示。这种序列可以保证每个状态切换时，只有一相桥臂的开关动作。

其中， $T_0 = T_7 = \frac{1}{2} (1 - T_4 - T_6)$

### 矢量出现顺序和其他扇区的开关序列
不管是七段法还是五段法，其矢量出现的顺序先后都按照下图所示：

<image src="矢量出现顺序.jpg">

然后再根据到底是七段还是五段决定后面的状态是怎样的。

七段法和五段法的开关序列见下表：

<image src="七段和五段开关序列.jpg">

在任意一个扇区，合成完一次参考电压向量后开关状态总是回到零向量位置 。参考电压向量从一个扇区转到了下一个扇区时，合成过程总是从一个零向量开始，这就保障了参考电压向量合成的连续性。

七段式三相在六个扇区的完整波形如下：

<image src="七段式完整波形1.jpg">
<image src="七段式完整波形2.jpg">
<image src="七段式完整波形3.jpg">


### 三段法、五段法、七段法优缺点

单个PWM周期内，5段式方法将零矢量集中插入在中间，转矩脉动大，在低频时会导致明显的走走停停不平稳现象，而7段式方法中零矢量的一半被插入在PWM周期的中间，另一半插入在PWM周期的两边，这样可以使得磁链的运转更加平稳，减少电机转矩的脉动，使得低频时特性明显好于5段式，高频时特性差异不大。但5段式方法中每个PWM周期中，总有一相桥臂的开关管状态不需要改变，而在7段式方法中，每一相桥臂的开关管都需要开关各一次，5段式比7段式开关次数减少1/3，所以5段式的开关功耗是最小的。


## 6.2 联系SVPWM和 $U_q$ $U_d$ $U_\alpha$ $U_\beta$

回顾扇区一中的做功时间表达式：

$T_4=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin(\frac{\pi}{3}–\theta)$

$T_6=\frac{2\sqrt3}{3}\frac{u_m}{V_{dc}}T_ssin\left(\theta\right)$

已知了电角度 $\theta$， 通过设置合成电压矢量$V_out$d的大小$u_m$进行电机的力矩控制。这里有个问题，$u_m$到底怎样设置？？

在第五章中，推导得到SVPWM的相电压为：

$$
U_a =\frac{2}{3}u_m \cos\theta
$$

也就是说相电压幅值是我们合成电压矢量大小的 $\frac{2}{3}$ 倍。所以参考等幅值Clark变化中的做法，一般会让

$$|V_{out}| = \sqrt{U^2_q+U^2_d} = \sqrt{U^2_\alpha+U^2_\beta} = \frac{2}{3} u_m$$

。这样，$|V_{out}|$ 就等于相电压的幅值了。

或者是在构建扇区图的时候，让$V_1 V_2 ... V_4 = \frac{2}{3}V_{dc}$ ， 效果一样，也是可以让$|V_{out}|$ 就等于相电压的幅值。

那么就有：

$$T_4= \sqrt{3} \frac{|V_{out}|}{V_{dc}}T_ssin(\frac{\pi}{3}–\theta)$$

$$T_6=\sqrt{3} \frac{|V_{out}|}{V_{dc}}T_ssin\left(\theta\right)$$

然后


***问题*** 为啥非要乘$\frac{2}{3}$让$U_q$ 等于相电压的幅值？这也没怎么方便啊，难道是为了和电流反馈中的等幅值Clark逆变换统一形式？

## 6.3 典型控制流程

<image src="svpwm力矩控制框图.jpg">

以这张图为例，该图是以电流闭环控制为例的，也就是让电机始终产生一个恒定的力矩（也就是恒定的电流，因为力矩和电流成正比）

从电流采样开始，采样的三相电流经过Clark逆变换后得到$I_\alpha$ 和 $I_\beta$ 。这里的Clark逆变换会采用等幅值的Clark逆变换，因为前面介绍过，等幅值Clark逆变换在计算上非常便利，而且$I_a$就直接等于$I_\alpha$。然后经过Park逆变换得到**真实**的$I_q$ 和 $I_d$。

**真实**的$I_q$ 和 $I_d$与参考的电流$I_qref$ 和 $I_dref \equiv 0$做差值，偏差送入PID里面，输出$U_q$ $U_d$ ，注意这里的$U_d$就不一定是0了，因为实际的$I_d$不一定是0。

之后经过逆park变换得到$U_\alpha$ 和 $U_\beta$，这一步主要是为了方便扇区判断，然后SVPWM输出pwm波形。

<image src="电流力矩控制.jpg">

电流环实际只用到了PI控制，没有引入微分，因为如果推导一下电压和电流的传递函数会发现这其实就是一个一阶惯性环节（而且实际上我们可以通过零极点对消来简化掉PI参数，只需要控制一个参数即电流带宽即可）


<image src="电流速度控制.jpg">

速度环同样只有PI控制，不用D。速度的微分就是加速度，就是力，就是电流，和电流的比例环是重复的。而且速度测量的噪声大，微分之后更是会放大噪声。

<image src="电流速度位置控制.jpg">

最外一层是位置环，位置控制PID只用了P项（也可以使用PI）

在实际使用中，由于编码器无法直接返回电机转速，因此可以通过计算一定时间内的编码值变化量来表示电机的转速（也即用平均速度代表瞬时速度）。当电机转速比较高的时候，这样的方式是可以的；但是在位置控制模式的时候，电机的转速会很慢，这时候用平均测速法会存在非常大的误差，尤其是三相霍尔传感器，磁编码器好一些。

所以为了避免速度环节带来的误差，在做位置控制的时候可以只使用位置和电流组成的双环进行控制，不过此时需要对位置环做一定的变化，控制框图如下：

<image src="电流位置控制.jpg">

由于去掉了速度环，这里的位置环我们使用完整的PID控制，即把微分项加上（因为位置的微分就是速度，这样可以减小位置控制的震荡加快收敛；积分项的作用是为了消除静态误差）。


# 6.4扇区判断



# 七、对$I_d$进行PID控制的意义

电机中实际产生磁场的是电流，然而我们只能控制端电压

之前的推导都是针对纯阻性的理想情况$I = \frac{U}{R}$ ，试图通过控制电压来控制电流。

然而，实际上电机是个感性器件，电流的变化是滞后端电压的，这就导致虽然我们试图让 $U_d = 0$ 但是实际的$I_d$其实不为0。也就是虽然我们试图让产生的磁场垂直于转子磁场，但是其实由于电流的滞后，始终是不严格垂直的。尤其是转速越高的时候，偏差越大，这导致效率和力矩下降。

对电流采样的意义是，直接对$I_q$和$I_d$进行反馈控制（当然还是通过电压控制电流）：让$I_q$始终是我们想要的大小，让$I_d$等于0。

DengFoc的电流环中只是对$I_q$进行了闭环控制，但是没有对$I_d$进行闭环控制，并没有发挥电流环的核心作用。




# 八、三次谐波注入