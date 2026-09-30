---
title: "Godot 开发笔记"
description: Godot 使用过程中的技巧与踩坑：Tool 脚本、padding、2D 动画、preload、存档、Theme、YSort、导航等
date: 2026-09-30T10:05:00+08:00
image: 
math: 
license: 
comments: true
draft: false
categories: ["Godot"]
tags: ["godot", "gamedev"]
build:
    list: always
---

## Tool引用非Tool节点

$Slot 挂的是 autom_slot.gd,那个脚本不是 @tool。编辑器里非 tool 脚本只是 placeholder 实例(只有底层 Area2D 的能力,脚本里声明的成员一律不存在)。

## padding
贴图如果没流出透明像素padding, 一些texture显示时由于uv的一些问题会导致边缘像素拉伸,比如下面的2d polygon
可以手动为texture的image加padding, 操作如下
```gdscript
func add_padding_to_texture(original_texture: Texture2D, padding: float) -> Texture2D:
	var oi = original_texture.get_image()
	if oi.is_compressed():
		oi.decompress()
	if oi.get_format() != Image.FORMAT_RGBA8:
		oi.convert(Image.FORMAT_RGBA8)
	var padding_i = Image.create(
		oi.get_width() + padding * 2,
		oi.get_height() + padding * 2,
		true,Image.FORMAT_RGBA8
	)
	padding_i.fill(Color.TRANSPARENT)
	padding_i.blit_rect(
		oi,Rect2i(0, 0, oi.get_width(),oi.get_height()),
		Vector2(padding, padding)
	)
	return ImageTexture.create_from_image(padding_i)

```
## 2d动画思路

**godot接入live2d三方sdk**
尝试中

**2d骨骼+ Sprite2D**
这种直接将精灵放到骨骼下, 纯利用骨骼的transform属性来进行动画, 而不对精灵本身有变形.
通常搭配ik反向动力学 + 控制点, 用于实现简单的ik动画, ik趋势骨骼,作为子节点的精灵跟着变换位置
*注意精灵本身不要使用position, 应该用offset进行调整*

**2d骨骼+ polygon2D**
这种建立骨骼, polygon2d绑定好纹理绘制好网格后放在骨骼之外.
将polygon2d和骨骼拼成默认姿态, 先设置骨骼的放松姿势, 再在polygon2d上给骨骼刷权重.
godot的2d骨骼权重工具非常拉...便利性不足需要大量手k
通过手k网格和骨骼权重从而实现形变效果. 然而这样的效果也只是限制在骨骼的上限内,对于2d来说灵活性并不高.

*注意精灵本身不要使用position, 应该用offset进行调整*



## 坑:preload导致嵌套
gd中写preload, 如果gd自身也在preload的场景中,则会造成嵌套.
嵌套并非不可用,**godot对嵌套的节点会进行延迟生成**, 而这会导致一些节点的ready,本该获取到的node引用由于延迟生成而为空.
报错只会提示你引用为空,而不会告诉你因为嵌套延迟生成.

因此注意排查和谨慎使用preload避免嵌套引用.

>  同样: TileSet使用场景图块时要求场景不能引用 "引用这个场景自身的" TileSet, 不然会导致实例化该场景时找不到这个TileSet 

## 新的保存思路
组件化  + Resource.
这里使用Resource创建NodeSaveData基类来处理不同类型的SaveData, 通过继承NodeSaveData创建各种保存数据, 同时附加_save_data 方法来接收node获取保存数据.
基础为: 
global position, node_path , parent_path,

在SaveComponent中持有NodeSaveData, 并在save_data方法中调用持有的NodeSaveData对象的_save_data方法.

对于Scene实例化的节点, 或者tilemap等有自己load逻辑节点, 提供的resource的_load_data与_save_data相对
- scene会获取node的scene file path并存储, load时使用路径再次实例化
- tilemap会记录下tilemaplayer的瓦片信息在load时还原瓦片

SceneLevelComponent负责游戏存储和导入. 其save_game方法获取所有save_component组的component node,调用save_data方法从而利用多态保存, 复制这些组件的save_data存入数组中, 最后保存到SecenLevelSaveData实例,并序列化到game_data 文件夹下文件里.

>  注意此处路径使用godot提供的用户数据路径. window上为appdata 下

## 输入映射

**tips**
- 组合键action先判断 is_action_pressed, 因为组合键触发时单键的action也可能触发,因此要提高组合键判断优先级

## Theme

godot的主体功能非常完备.
theme概念完全抽象且优先. 你可以创建theme,theme能保存为资源并提供各种空间类型的样式.

祖先控件的主体会覆盖影响子级的默认样式

其次,主题中可以制作自定义类型并命名, 自定义类型可以继承基础类型,在自定义类型中设置样式.
使用时只要在主题影响下的控件上设置主题类型变体为自定义类型名称就可以应用了
注意控件的类型要匹配自定义类型的基类.

>  想想Theme的功能完全就是css选择器的一套封装



## Tile:
godot的瓦片系统亦如其他瓦片系统,
TileSet对象支持建立图集, 作为实际绘制瓦片时的调色板
图集分为:
- Anim
- Terrain 地形
- ...
TileMapLayer则负责按层绘制瓦片图集
其需要一个TileSet

- 编译godot源码了,感觉godot的cpp源码也值得阅读啊,scene 下的组件实现有部分常用的游戏逻辑
	- scons感觉好爽哦吼吼

TileSet支持使用场景作为图块
TileSet支持绘制图块碰撞

## 排序遮挡问题
Godot2D 使用画家算法绘制,渲染顺序是根据节点顺序与z_index控制的.
对于TileMapLayer, 我们无法这么做,因为其与player的绘制顺序并不固定而是根据position决定的.
player.y < a wall.position.y时,玩家在墙后应该被遮住.反之墙被遮住.
此时需要启用canva item的y sort

简单来说，父 `YSort` 会收集所有**直接子节点**（无论它们是不是 `YSort`），然后根据这些子节点的全局 Y 坐标统一排序, Y 坐标值大的节点会绘制在更上层。(要求参与排序的zindex相同)
### 🧩 那么，`YSort` 子节点内部的“孙节点”呢？

这是行为最关键的差异点：
- **非 `YSort` 子节点**：其内部的子节点（孙节点）**不会**参与父级的 YSort 排序，也不会被父级 YSort 单独处理。这些孙节点通常会作为一个整体，跟随它们的父节点参与排序。
- **`YSort` 子节点**：YSort节点可以嵌套。**子YSort节点将与父节点在相同的空间内进行排序**，这样可以更好地组织一个场景或将其分为多个场景，但又能保持唯一的排序。

在TileSet中,你可以设置图块的YSort 原点来控制其排序效果

**Tips**
- AnimatedSprite2D 的ysort是根据自己的position进行判断, 不考虑offset,因此调整图片动画位置使用offset同时保证position在脚部以符合逻辑
## 导航
使用navigation Region 2D划定导航区域
使用Navigation Agent2D 驱动角色进行导航
Navigation layer要求设置为同一层

用法为
```gdscript
func _ready():
    # 告诉代理要去哪（例如，跟随玩家）
    navigation_agent.target_position = get_global_mouse_position() # 或其他目标位置

func _physics_process(delta):
    # 检查是否已经到达终点，避免在终点处抖动
    if navigation_agent.is_navigation_finished():
        return
    # 1. 获取下一个路径点
    var next_position = navigation_agent.get_next_path_position()
    
    # 2. 计算移动方向
    var direction = (next_position - global_position).normalized()
    
    # 3. 移动你的角色
    velocity = direction * SPEED
    move_and_slide()
```
Agent之间的避障则需要开启Avoidance, 并由navigation agent代理.
使用时,  向agent设置理论速度, agent计算安全避障速度,并在一个回调中返回给你计算结果的安全速度.

# 入门
最近开始做我那个解释器 + 字符拼接语言玩法的解密游戏了.
搭建核心玩法 demo 中..

不过自己引擎经验好少, 对于脚本逻辑写的不是很顺畅, 而且 gdscript 也要重新学.

感觉需要多看文档理解基础, 语言基础. 还有常见做法和设计
几个方向吧
