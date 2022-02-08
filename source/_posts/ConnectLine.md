---
title: 连接线
date: 2022-02-03 20:11:43
tags: 作品集
math: true
---
- 连接线可用于场景中两个点之间的指向表示

***

# 材质
## 透明度输入
### 中间箭头
#### 采样用图
- <img src = '/ConnectLine/Arrow.png' width='30%'>
- 由于涉及纹理动画，贴图应保留一定的出血，且寻址方式要设为Clamp
- 为防止据相机较远时的采样问题，设置不使用Mipmap
- 采样结果：<img src = '/ConnectLine/InitSample.png' width='30%'>

#### 缩放
- 设钳制在[0, 1]的缩放系数**ScaleLevel**，令`UV \= ScaleLevel`以缩放UV并重新采样：<img src = '/ConnectLine/UVScale.png' width='30%'>

#### V方向偏移
- 缩放后为保证箭头仍位于V方向的中间，需进行偏移
- 缩放后UV在V方向的最大值为`VMax = 1.0 / ScaleLevel`，则应该将箭头向下移动`VMax / 2 - 0.5`个单位，即V方向的偏移量为`VOffset = -(VMax / 2 - 0.5)`：<img src = '/ConnectLine/VOffset.png' width='30%'>

#### U方向偏移
- 为表示连接线的指向性，需要使箭头在U方向进行纹理动画
- 缩放后UV在U方向的最大值为`UMax = 1.0 / ScaleLevel`，则U方向箭头的移动范围为`[-1, UMax]`
- 为控制箭头的移动速度，则利用`NormalizedTime = frac(Time * MovSpeed)`做循环动画
- 综上，得到U方向的偏移量为`UOffset = NormalizedTime * -(UMax + 1) + 1`
- 偏移后的结果：<img src = '/ConnectLine/UOffset.gif' width='30%'>

### 两侧线段
#### 采样用图
- 两侧线段使用一张Ramp图即可，方便控制：<img src = '/ConnectLine/ArrowRamp.png' width='30%'>
- 采样结果：<img src = '/ConnectLine/RampSample.png' width='30%'>

#### 旋转
- UE中实现该效果时，习惯以X轴方向为前向，故需要将Ramp图逆时针旋转90°
- 这里利用二维旋转矩阵对UV进行变换，即$\dbinom{U'}{V'} = \begin{bmatrix} cos\alpha & -sin\alpha \\ sin\alpha & cos\alpha \end{bmatrix} \dbinom{U}{V}$
- 由于我们需要以UV中心旋转，所以需要先令`UV -= float2(0.5, 0.5)`使移动到中心处，然后对U和V分量分别进行点乘以完成上述的矩阵乘法操作，最后再令`UV += float2(0.5, 0.5)`移回原位
- **旋转前应进行所需的缩放和平移操作**，这里先展示单独旋转的结果：<img src = '/ConnectLine/RampRot.png' width='30%'>

#### V方向缩放
- 两侧线段由宽变窄，首先通过缩放来实现该效果
- 设线段末端宽度占比为`EndPercent`∈[0, 1]
- 得到V方向缩放量`VScale = UV.x * (EndPercent - 1) + 1`，使得缩放由1变为EndPercent
- 令`UV \= float2(1.0, VScale)`以缩放UV：<img src = '/ConnectLine/RampScale.png' width='30%'>

#### V方向偏移
- 对线段在V方向进行偏移以修正缩放的结果
- 与箭头的偏移处理相似，由于缩放的最大值为`VMax = 1.0 / VScale`，故偏移量为`VOffset = -(VMax / 2 - 0.5)`
- 令`UV += float2(0, VOffset)`：<img src = '/ConnectLine/RampOffset.png' width='30%'>
  
### V方向透明度渐变
#### 箭头和两侧线段的叠加
- 将上述箭头和线段的结算结果做加法：<img src = '/ConnectLine/ArrowRampAdd.gif' width='30%'>

#### 渐变
- 为防止连接线两端过于生硬，添加渐变效果；这里利用两个SmoothStep相减的方式来实现：<img src = '/ConnectLine/VGradient.png' width='30%'>
- 将渐变与之前的叠加结果相乘：<img src = '/ConnectLine/ArrowRampAddGradient.gif' width='30%'>

## 颜色输入
- 将所需要设置的颜色乘上明度控制以及饱和度控制，作为自发光颜色即可
- 结合之前的透明度输入得到结果：<img src = '/ConnectLine/ColorInput.gif' width='30%'>

## 顶点偏移输入
- 连接线应为起点和终点之间的一条曲线，这里使用抛物线来表示
- 这里应用的抛物线公式为$f(x) = \frac{(x-a)^2-a^2}{-2b}$，其中a应设置为起终点距离的一半；b控制抛物线的弯曲程度
- 由于连接线后续通过缩放进行延长，因此这里的a为初始Mesh的尺寸的一半，如可设为50个单位长度
- b可设置为`1.0 / BendLevel`，**BendLevel**可设为弯曲度的调整参数
- x可设置为每个顶点的局部坐标；最后得到的f(x)作为z分量应用在顶点偏移即可
- 调整**BendLevel**的结果：<img src = '/ConnectLine/BendLevel.gif' width='50%'>

***

# 模型
- 基础模型使用一张UE中100*100单位大小的平面即可，但由于需要弯曲，因此需要在X方向上进行细分；令由于需配合材质中顶点输入的抛物线计算，将模型的轴心设为一侧的中点
- <img src = '/ConnectLine/ArrowPlane.png' width='30%'>

***

# 蓝图
- <img src = '/ConnectLine/BP.png' width='100%'>
- 其中ArrowHeight为模型的宽度，一般设为100个单位；Aspect为连接线的长宽比

***

# 最终效果
<img src = '/ConnectLine/Result.gif' width='100%'>