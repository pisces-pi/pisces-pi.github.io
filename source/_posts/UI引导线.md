---
title: UI引导线
date: 2022-02-06 22:41:45
tags: 作品集
math: true
hide: true
---
- UI引导线是存在于屏幕空间的、由指定的起点指向终点的线段
- 其中构成的线段的图案可自由配置

***

# 材质
## 图案
- 使用一张边缘亮度高于中间的箭头图案：<img src = '/UI引导线/GuideLinePattern.png' width='30%'>
- 图案设置为Clamp的寻址方式；且不使用Mipmap
- 采样结果：<img src = '/UI引导线/PatternSample.png' width='30%'>

## 旋转
- 定义**StartPoX**、**StartPosY**和**RotateAngle**三个参数。其中前两者为引导线在屏幕上的标准化坐标；后者是为使引导线由起点指向终点所需旋转的角度
- 使用旋转矩阵对UV进行旋转，旋转中心即为`float2(StartPoX, StartPosY)`，旋转角度为`RotateAngle`
- 得到结果(绕(0.7, 0.7)旋转45°为例)：<img src = '/UI引导线/Rotate.png' width='30%'>

## U方向范围计算
- 旋转之后，U方向的分布为：<img src = '/UI引导线/RotateU.png' width='30%'>
- 通常来说，引导线不应该占据整个屏幕空间，且应允许配置其宽度
- 定义一个参数**Width**表示线段的宽度
- 计算U方向的范围遮罩：
  ```
  float halfWidth = Width / 2;  //宽度的一半
  float rangeStart = max(StartPosX - halfWidth, 0.0);  //范围起点
  float rangeEnd = max(StartPosX + halfWidth, 0.0);  //范围终点
  float startMask = saturate(floor(UV.x / rangeStart));  //起点遮罩
  float endMask = saturate(floor(UV.x / rangeEnd));  //终点遮罩
  return startMask - endMask;
  ```
  起点遮罩减去终点遮罩：

<center>
  <img src = '/UI引导线/StartMask.png' width='20%'> -
  <img src = '/UI引导线/EndMask.png' width='20%'> =
  <img src = '/UI引导线/URange.png' width='20%'>
</center>

- 再计算渐变区域：
  先令`UV.x -= rangeStart(上方代码块中的临时变量)`，使0-1值域位于所需范围：<img src = '/UI引导线/GradientRange.png' width='30%'>
  
  再令`UV.x = frac(UV.x / Width)`，得到周期性的0-1范围：<img src = '/UI引导线/UFracResult.png' width='30%'>

  最后将遮罩和渐变区域相乘得到所需的U方向范围：

<center>
  <img src = '/UI引导线/URange.png' width='20%'> *
  <img src = '/UI引导线/UFracResult.png' width='20%'> =
  <img src = '/UI引导线/UGradientRange.png' width='20%'>
</center>

## V方向范围计算
- 旋转之后，V方向的分布为：<img src = '/UI引导线/RotateV.png' width='30%'>
- 设引导线的长度为**TotalDistance**；我们期望图案在终点处保持完整，而在起点处可以被截断，故首先在V方向上偏移，使V向上的0值位于终点处：`UV.y = max(UV.y + (TotalDistance - StartPosY), 0)`：<img src = '/UI引导线/VOffset.png' width='30%'>
- 之后根据宽度进行frac操作以实现Tilling：`frac(UV.y / Width)`：<img src = '/UI引导线/VFracResult.png' width='30%'>
- 另计算用于截断的遮罩：`saturate(floor((1.0 - UV.y) / max((1.0 - StartPosY), 0)))`：<img src = '/UI引导线/VMask.png' width='30%'>
- 最后将遮罩和之前的Tilling后的值相乘以得到所需的V向范围：

<center>
  <img src = '/UI引导线/VFracResult.png' width='20%'> *
  <img src = '/UI引导线/VMask.png' width='20%'> =
  <img src = '/UI引导线/VRange.png' width='20%'>
</center>

## 构造UV
- 首先确定好V方向的值，为：

<center>
  <img src = '/UI引导线/VRange.png' width='20%'> *
  <img src = '/UI引导线/URange.png' width='20%'> =
  <img src = '/UI引导线/ResultV.png' width='20%'>
</center>

- 之后根据得到的V向的值，对其使用`ceil`操作上取整得到一个遮罩，再去乘上原来的U向的值：

<center>
  <img src = '/UI引导线/UGradientRange.png' width='20%'> * $\Biggl($
  <img src = '/UI引导线/ResultV.png' width='20%'> $\xrightarrow{ceil}$
  <img src = '/UI引导线/VCeil.png' width='20%'> $\Biggr)$
  <img src = '/UI引导线/ResultU.png' width='20%'>
</center>

- 最后构造出所需的UV：

<center>
  append $\Biggl($ <img src = '/UI引导线/ResultU.png' width='20%'> ,
  <img src = '/UI引导线/ResultV.png' width='20%'> $\Biggr)$ =
  <img src = '/UI引导线/ResultUV.png' width='20%'>
</center>

- 此时采样乘上一个颜色的结果：

<img src = '/UI引导线/InitSample.png' width='30%'>

## 弯曲
### 获取V向的0-1范围
- 使引导线弯曲以添加更加丰富的效果，这里使用一个抛物线来实现
- 首先需要得到一个灰度范围，由终点处的0值线性插值到起点处的1值
- 以旋转后的UV的V向作为计算输入，令`UV.y = max(UV.y + (TotalDistance - StartPosY), 0.0)`，使得0值位于线段的终点处；此时V向的值域为$[0, 1 + (TotalDistance - StartPosY)]$
- 再令`UV.y /= 1 + (TotalDistance - StartPosY)`，使V向的值位于0-1之间
- 最后令`UV.y = min(UV.y / StartPosY, 1.0)`，使得线段起点的V向值为1
- 得到：<img src = '/UI引导线/InitBend.png' width='30%'>

### 利用抛物线实现弯曲
- 由于我们需要的是线段中心的弯曲程度最大，所以需要一个以x = 0.5对称的抛物线，且在横坐标0和1处的值为0
- 易得公式为：$f(x) = bend * x * (1 - x)$，其中$Bend$为弯曲程度，可定义为**BendLevel**
- 对之前得到的结果进行操作：`Bend = BendLevel * UV.y * (1.0 - UV.y)`
- 最后，使旋转过的`UV.x += Bend`，再按上文所述构造UV，得到：<img src = '/UI引导线/BendUV.png' width='30%'>
- 进行图案采样的结果：

<img src = '/UI引导线/BendUVSample.png' width='30%'>

## 半透明
- 引导线可添加由起点到终点的完全透明到不透明的渐变过渡效果
- 令`pow(1 - UV.y, OpacityRange)`，其中的UV.y为上文中最终得到的**V向值域为0-1间的值**以得到一个渐变区域；其中**OpacityRange**用于控制半透明的范围：<img src = '/UI引导线/OpacityRange.png' width='30%'>
- 采样结果：

<img src = '/UI引导线/OpacitySample.png' width='30%'>

## 动效
### 缓出移动
- 引导线的第一个动效为由起点移至终点的缓出效果
- 这里使用一个简单的缓出公式：$f(x) = 2x - x^2$，其中x为$0\rightarrow1$的变量，可先通过`frac(Time)`传入，即
  ```
  float fracTime = frac(Time);
  float offsetTime = 2 * fracTime - pow(fracTime, 2.0);
  ```
- 在之前计算V方向范围时，我们利用`UV.y = max(UV.y + (TotalDistance - StartPosY), 0)`使UV在V向上平移；这里我们先令`TotalDistance *= offsetTime`，使`TotalDistance`由0变化为最终值，以实现缓出的移动效果：<img src = '/UI引导线/EaseOut.gif' width='30%'>
- 最终采样结果：

<img src = '/UI引导线/EaseOutResult.gif' width='30%'>

### 消隐
- 引导线的第二个动效是令引导线逐渐消失
- 实现只需在之前的半透明区域基础上乘上一个随时间由$1\rightarrow0$的值(`1 - frac(Time)`)即可：

<center>
  <img src = '/UI引导线/OpacityFade.gif' width='20%'> *
  <img src = '/UI引导线/OpacityRange.png' width='20%'> =
  <img src = '/UI引导线/OpacityRangeFade.gif' width='20%'>
</center>

- 最终效果：

<img src = '/UI引导线/OpacityFadeResult.gif' width='30%'>

### 动效结合
- 需要将上述的两个动效结合起来，先进行缓出，再进行消隐
- 设两个变量**OffsetSpeed**和**FadeSpeed**，分别表示缓出的速度和消隐的速度
- 进行计算：
  ```
  float offsetTime = 1.0 / OffsetSpeed;
  float fadeTime = 1.0 / FadeSpeed;
  float totalTime = offsetTime + fadeTime;
  float totalSpeed = 1.0 / totalTime;

  float offsetRatio = offsetTime / totalTime;  //偏移占总时间的比例
  float fadeRatio = fadeTime / totalTime;  //消隐占总时间的比例

  float fracTime = frac(Time * TotalSpeed);  //最后由时间传出的值域控制在0-1
  float time4Offset = saturate(fracTime / offsetRatio);  //时间的前部分用于进行偏移
  float time4Fade = saturate((fracTime - offsetRatio) / fadeRatio);  //时间的后部分用于消隐
  ```
- 将上述计算得到的**time4Offset**和**time4Fade**分别应用在偏移和消隐上即可：
  <img src = '/UI引导线/DynResult.gif' width='30%'>

## 屏幕适配
- 由于UI引导线存在于屏幕空间，因此需要根据屏幕大小进行适配
- 以16:9的屏幕长宽比为例，此时直接采样的结果：<img src = '/UI引导线/PreScreenAdapt.png' width='30%'>
- 定义一个参数Aspect，表示屏幕的长款比；首先将它作为U方向的Tilling倍数，在16:9的情况下值即为$\frac{16}{9}$
- 对于本例来说，只需令所有的`StartPosX *= Aspect`即可：<img src = '/UI引导线/ScreenAdapt.png' width='30%'>