---
layout: post
title: "基本元件特性"
math: true
categories: 电路
tags: 电路设计
author: fengsh998
typora-root-url: ..
---

### 电阻
电路符号：R，无极性

**欧姆定律**

![img](/assets/articles/ic/elements/om.jpg)

$$ V = I⋅R $$

* V：电阻两端电压 (V)
* I：流过电阻的电流 (A)
* R：电阻阻值 (Ω)

特性作用：

* 限流保护、分压、偏置

### 电感
电路符号：L，无极性

![img](/assets/articles/ic/elements/lll.jpg)

电感两端的电压取决于电流变化的快慢。
* 如果电流恒定不变 ($ \frac{di}{dt} =0 $)，则电感表现为短路（理想情况下电压为0）。
* 如果电流变化极快（如脉冲信号），电感会产生巨大的感应电动势。

**法拉第电磁感应定律**

$$ V = L⋅ \frac{di}{dt} $$

* V：电感两端的电压（单位：伏特 V）。
* L：电感量（单位：亨利 H）。
* $ \frac{di}{dt} $：电流对时间的变化率（单位：安培/秒 A/s）。
  
电流变化率 (di/dt) 示意: 当电路开关闭合时，电流不会瞬间跳变，而是按指数规律上升，电感起到了“缓冲”作用。

感抗： $ X_L = 2πfL $

表现为：电压超前于电流  

看到电流波动需要平稳，优先考虑 电感。

**特性：** 
* **通直流，阻交流**：直流电容易通过电感（相当于导线），而交流电通过时会产生感抗，从而衰减交流信号。
* **储能（升压）**：（磁场能）
* **滤波（平滑电流）**。能有效抑制电流的剧烈波动，保持电流稳定。
* **谐振**
* **电流不能突变**：当流过电感的电流发生变化时，会产生反电动势来抵抗这种变化。

#### 电流的变化规律及达到“最大值”的时间

直流电源5V，负载R=1K，电感L= 1H，求电流最大时的爬升时间？
在电路合闸瞬间，电感相当于“断路”（电流为 0），电流从 0 开始增长。公式为：

<font color=red>$$ i(t) = \frac{V}{R}(1-e^{-\frac{R}{L}t}) $$</font>

公式推导：电感电路的瞬态分析基于基尔霍夫电压定律（KVL）：在回路中，电源电压等于电感电压与电阻电压之和。

$$ V = V_L + V_R = L\cdot\frac{di}{dt} + iR $$

① 整理后得： $ \frac{di}{dt} = \frac{V-iR}{L} = -\frac{R}{L}(i-\frac{V}{R}) $

② 移项：$ \frac{di}{i-\frac{V}{R}} = -\frac{R}{L}dt $

③ 两边同时积分： $ \int\frac{1}{i-\frac{V}{R}}di = \int-\frac{R}{L}dt $

根据概念：

核心积分公式: $ \int\frac{1}{u}du = ln|u| + C $
在微积分中，有一个基础规则：任何形如 $ \frac{1}{x} $ 的函数的原函数（不定积分），就是 ln∣x∣。

因此 $ \int\frac{1}{i-\frac{V}{R}}di $ 令 $ u = i - \frac{V}{R} $ 
那么，对u求导则得到 $ d(u) = d(i - \frac{V}{R}) => du = di - d(\frac{V}{R}) $ ,因任何常数的导数为0所以，最终得到 du = di。 

侧积分的左边：$ \int\frac{1}{i-\frac{V}{R}}di = \int\frac{1}{u}di $ 

$ \int\frac{1}{u}du = ln|u| + C $ 。 

将u代入，最终左边变为：$ ln(i - \frac{V}{R}) + C $

现在再来看右边的不定积分： $ \int-\frac{R}{L}dt $

根据 $ \int k \cdot f(t)dt=k \cdot \int f(t)dt $ ,这里 $ k = -\frac{R}{L} , f(t) = 1 $

所以变为 $ -\frac{R}{L}\cdot\int1dt $ , 常数的积分 $ \int1dt = t +C $

最终右边变换为 $ -\frac{R}{L}\cdot(t+C) $

到些原积分最终变为：

$$ \int\frac{1}{i-\frac{V}{R}}di = \int-\frac{R}{L}dt  =>  ln(i - \frac{V}{R}) + C = -\frac{R}{L}\cdot(t+C) $$

④ 精简后得到：

  $$ ln(i - \frac{V}{R}) = -\frac{R}{L}t + C $$

第一步：消去 ln（取指数），为了解出 i，我们需要把左边的 ln 去掉。两边同时以 e 为底取指数：

 $$ e^{ln(i - \frac{V}{R})} = e^{-\frac{R}{L}t + C} $$
 
 根据对数性质，$ e^{ln(x)} = x $ ，所以左边直接变成了 $ (i - \frac{V}{R}) $
 
 右边利用幂指数法则 $ e^{a + b} = e^a \cdot e^b $，变为：
 
 $$ i - \frac{V}{R} = e^C \cdot e^{-\frac{R}{L}t} $$
 
 第二步：处理常数项
 
 由于 e 是一个常数，C 是一个常数，$ e^C $ 也是一个常数。为了书写方便，我们令 $ A = e^C $ ，公式变为：
 
 $$ i - \frac{V}{R} = A \cdot e^{-\frac{R}{L}t}  => i(t) = \frac{V}{R} + A \cdot e^{-\frac{R}{L}t} $$
 
 第三步： 利用初始条件求出A， 在 t=0 的那一刻，电路刚合闸，电感还没来得及充入电流，所以 i(0)=0。
把 t=0 和 i=0 代入上面的方程:

$$ 0 = \frac{V}{R} + A \cdot e^{-\frac{R}{L} \cdot 0} $$

因为  $ e^0 = 1 $ , $ 0 = \frac{V}{R} + A \cdot 1 $ ,解得 $ A = -\frac{V}{R} $

第四步：代回原方程 $ i(t) = \frac{V}{R} + (-\frac{V}{R} \cdot e^{-\frac{R}{L}t}) $

终得出上面的公式。

* * 当最流最大时($ I_max $): 当$ t -> \infty $ 时, $ i(t) = \frac{V}{R} = \frac{5V}{1000\Omega} = 5mA $
* * 时间常数($ \tau $):衡量电路响应速度的物理量,$ \tau = \frac{L}{R} = \frac{1H}{1000\Omega} = 0.001s = 1ms $
   计算结果： t=5×1ms=5ms。
   
* * 当电感达到稳定态时，这时就和导线一样，电流最大此时理论最大的电流为 $ I_max = \frac{V}{R} $ 最后关系式为： $ i(t) = I_max(1 - e^{-\frac{t}{\tau}}) $

因为指数函数 $ e^{−x} $ 是随着 x 增大而迅速衰减的。$ e^{−\frac{t}{\tau}} $ 中的t是在指数位置。要

想Imax最大那 $ e^{-\frac{t}{\tau}} $ 无限逼近于0，即 $ e^{-\frac{R}{L}t} \lim -> 0 $

### 电容
电路符号：C，有极/无极性

**基本方程**

$$ I=C⋅\frac{dv}{dt} $$

* I：流过电容的电流（单位：安培 A）。
* C：电容量（单位：法拉 F）。
* $ \frac{dv}{dt} $：电压对时间的变化率（单位：伏特/秒 V/s）。

看到电压波动需要平稳，优先考虑 电容。

容抗：$ X_c = \frac{1}{2πfC} $

表现为：电流超前于电压 

**特性：**

* **隔直流，通交流：** 由于电容两端电压不能突变，直流电在充电完成后，电容表现为开路状态。
* **储能与滤波（电场）：** 将电能以电场的形式存储在两个极板之间。充电时储存电能，放电时释放能量，可平滑电源纹波，去除高频杂讯。
* **旁路与去耦 (Bypass & Decoupling)**。
* **信号耦合 (Coupling)**
* **定时与延时**。
* **电压不能突变：** 当电路电压改变时，电容器两端的电压变化需要充电过程，常用于平滑电路中的电压波动。

![img](/assets/articles/ic/elements/licensed-image.jpeg)