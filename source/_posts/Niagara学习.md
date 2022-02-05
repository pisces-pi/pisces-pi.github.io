---
title: Niagara学习
date: 2022-01-23 20:35:21
---
# Niagara学习
## 层级关系
- 系统->发射器->粒子
- 组->模块
## 系统
### 组
- System Settings 设置
- System Spawn 生成
- System Update 更新
## 发射器
- 发射器以堆栈运行，即从上到下运行每个组的内容
- 左上角复选框：该发射器是否参与运算
- 小人图标：孤立当前发射器，其他发射器隐藏
- 小人右侧图标：显示了目前的渲染器(renderer)
- 节点上右键可以控制选中节点的启用和孤立设置
### 组 
- Emitter Settings 设置
- Emitter Spawn 发射器生成
- Emitter Update 发射器更新
- Particle Spawn 粒子生成
- Particle Update 粒子更新
- Add Event Handler 添加事件
- Render 渲染
### 循环与发射状态
- 在Emitter Update组的Emitter State模块
- Inactive Responce决定了发射器的循环
## 继承
- 对于继承操作，若继承的是来自模板的发射器，则可以进行对继承下来的模块编辑；否则不可以
- 同理，由非模板的发射器构建的系统里也不能直接修改继承的发射器模块
- 被继承的发射器在父发射器的模块被修改时将同步更改自己的模块
- 发射器的Asset Options->Is Template Asset被勾选后，即可变为一个模板资产
- 对于系统中使用的模板发射器，若后续将该发射器变为非模板，它仍可在系统中被修改模块；除非发射器被重新创建
## TimeLine
- TimeLine上的项可以展开，然后可通过类似打关键帧的方式添加burst的spawn模块