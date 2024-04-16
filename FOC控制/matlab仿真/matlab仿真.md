

<image src="三相PWM转换为互补输出.png">

NOT模块默认情况下 输入数据类型和输出数据类型可以不一致，而且输出是bool类型。

<image src="NOT模块参数.png">

虽然我们前面经过relay模块得出的是一个 0/1信号，但是数据类型是double，经过NOT模块变成了bool类型，需要再过一个double模块，这样最后输出的互补信号数据类型就是相同的了，都是double

<image src="NOT后面接Double模块.png">
