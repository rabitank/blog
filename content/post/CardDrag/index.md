---
title: "CardDrag"
description: 
date: 2026-07-28T14:11:43+08:00
image: 
math: 
license: 
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list

---

## 

## 前情提要

想要学习UI处理和工程经验观看了

[阿尔的秘宝](https://space.bilibili.com/24830836/?spm_id_from=333.788.upinfo.detail.click) 的 [卡牌游戏开发系列](https://www.bilibili.com/video/BV1EwEwzvEv4)

## 

## 拖拽

 卡牌拖拽核心代码: 弹簧和阻尼模拟.

```gdscript
@export var card_current_state = CardState.following
@export var following_target: CanvasItem = null
@export var damping: float = 0.21 # 阻尼
@export var stiffness: float = 140 # 弹力系数
@export var velocity: Vector2 = Vector2.ZERO
# Called every frame. 'delta' is the elapsed time since the previous frame.
func _process(delta: float) -> void:
    match card_current_state:
        CardState.dragging:
            var target_pos = get_global_mouse_position() - size / 2
            global_position = global_position.lerp(target_pos, 0.4)
        CardState.following:
            if following_target != null :
                var target_pos = following_target.global_position
                var displacement = target_pos - global_position
                var force = displacement * stiffness 
                velocity += force * delta
                velocity *= (1 - damping) # 阻尼
                global_position += velocity * delta

            if Input.is_mouse_button_pressed(MOUSE_BUTTON_LEFT):
                self.card_current_state = CardState.dragging
```

## Csv卡牌数据

csv格式读取辅助函数,读取卡牌信息

```gdscript
func read_csv_as_nested_dict(path: String) -> Dictionary:
    var data = {}
    var file := FileAccess.open(path, FileAccess.READ)
    var headers  = []
    var first_line = true
    while not file.eof_reached():
        var values = file.get_csv_line()
        if first_line:
            headers = values
            first_line = false
        elif values.size() >= 2:
            var key = values[0]
            var row_dict = {}
            for i in range(0, headers.size()):
                row_dict[headers[i]] = values[i]
            data[key] = row_dict
    file.close()
    return data
```

## 手牌容器-deck

使用z_index + 节点位置进行排序标记, 使用position.x进行排序, 使其从左到右进行叠放.

deck核心容器为hbox, 使用card_background作为子节点紧密自动排列, card_background如我注释所写, 实际为大小和card相同的占位节点,因此card_background作为follow_target,其变换遵循hbox排列.

当card_background发生顺序改变时,切换的新位置,其following的card会因此缓动跟随到新位置,因此就实现了整副手牌的丝滑缓动变换效果.

> 这里很有意思的是更换父子级时直接缓存global_position在换完后赋值回去,以保持位置不变,惹啊自己之前想的好复杂.

```gdscript
func sort_nodes_by_position(children: Array):
    children.sort_custom(sort_by_position)

func add_card(card_to_add: Card) -> void:
    var index = card_to_add.z_index
    # card_background 类似于deck中的占位符,大小和card相同, 作为follow_target
    var card_background = preload("uid://573356y5wf24").instantiate()
    card_deck.add_child(card_background)

    if index < card_deck.get_child_count():
        card_deck.move_child(card_background, index)
    else:
        card_deck.move_child(card_background, -1)
    var global_pos = card_to_add.global_position
    if card_to_add.get_parent():
        card_to_add.get_parent().remove_child(card_to_add)
    card_deck.add_child(card_to_add)
    card_to_add.global_position = global_pos
    card_to_add.following_target = card_background
    card_to_add.pre_deck = self

func sort_by_position(l, r):
    return l.position.x < r.position.x
```

### 手牌压缩

效果为所有卡牌间距变小, 紧贴在一起

实现方法, 利用deck的子节点们,也就是follow_target, 遍历将其custom_min_size设置变小,这样hbox会重布局,而card尺寸不变继续跟随它们各自target的新位置,因此跟随重布局的target聚在一起.



## 绘制效果-VfsLayer

VfsLayer: 一个Canvaslayer, 用于在拖拽时将卡牌的绘制层级提到最高以避免遮挡

拖拽控制: 考虑到绘制层级,拖拽发生时以及其他移动效果, 直接复制卡牌并挂到VfsLayer下用于动画. 

当然对于复制出来的卡牌记得标记为CardState.Vfs,在动画结束或者进入位置后销毁

## 锚点

总之是个UI知识点

卡牌牌面元素,利用锚点(父控件百分比位置) 加anchor offset(像素单位的锚点偏移或边距概念),可以控制控件矩形随着父控件矩形变化.

将锚点设置到父控件四边,自己通过anchor offset(红点)控制边距.

也因此建议对于子控件都按锚点方法设置,而顶层控件使用固定尺寸.

> 类似于web里的边距模型,但是web中无法控制锚点,只能是到父控件边缘的[无, px, 百分比] 三者. 
> 
> 而godot的模型中(我想UE应该也是这样),锚点是可以直接设置的, 因此实际上到父控件边缘配置为 百分比 + px. 这种组合比web上限更高.

## 存档

此处的存档中, card保存是直接将Node进行packedscene打包(pack方法) (因为scene也能直接被序列化?).好直接好爽.

packedScene打包方法对于场景树中的节点要求其节点的owner属性为场景根节点,因此动态添加的节点除非手动设置过否则不能自动进入打包的场景里.

因此这里还是使用了手动方式, 定义嵌套的字典/字段对象, 将手牌作为场景打包,再塞到自动加载对象里.

而对于存档对象中字典字段的key值,则使用了node.get_path()通过获取独一无二(人为)的path(每个level 场景的根节点使用不同的名字),进行存储.

存档对象则利用了resource类型,因为godot对资源对象的序列化支持比较完善和高自由度.

> resource要保存的字段必须@export

(但是据说并不适合存档因为默认值覆盖有问题,除此之外还有ini和json格式可用于存档)

需要保存的东西分为三类:

- 数值

- 打包好的场景(如card)

- 嵌套资源

怎么说呢,卡牌游戏中player就是一组纯数据,因此这里player直接就是resource并以此存档

>  游戏中的简化模型保持简单实际上类似于
> 
> 合并功能?比如这里就是player就是玩家,玩家就是存档
> 
> 实际上还有把子节点当作一种状态机状态的,非常直观.

## 读档

读档这里不必多说

使用

```gdscript
# Array[PackedScene]
for c in save.cards:
    var p = c.instantiate()
    add_card(p)
```

需要注意的就是将默认对象和需要始终存在的东西初始化



## UI遮挡

UI遮挡时一些事件无法传递,特别是如果需要事件的UI和遮挡它的Item不再同一个分支上时.

比如up这里制作的删除拖拽区域.然而其设计就只是需要鼠标是否进入按钮区域因此直接在process中获取鼠标位置进行判断.再手动发射mouse in / exited 信号



> process好用嗯艹



## 结束

后面的对话和商店由于和项目逻辑相关这里就不进行记录

总结就是:

- 丝滑效果秘诀:
  
  - 保持global_position不要有突变
    
    - 关注更换父级时位置
    
    - 利用follow_target间接控制位置
    
    - 使用基础的弹簧和阻尼模型来实现效果
  
  - 多多利用tween动画和渐入渐出曲线
  
  - 利用自带的h/vbox的布局功能控制follow_target

- 效果:
  
  - 利用VfsLayer来保证动画效果可见
  
  - 在需要时细调元素锚点

- 开发模式经验
  
  - 利用全局对象处理信息和游戏功能,大胆使用吧.
  
  - 获取对象时多多利用group来快速定位获得节点
  
  - 保存时可以利用pack scene来快速实现保存逻辑而不去思考保存数据设计
  
  - 嵌套对象,利用可见性来处理而不是生成节点,这样能够直接进行调整
  
  - ~~感觉没必要单独调试某些小场景的游戏逻辑,游戏逻辑都放到主场景开始测试就好了~~
  
  - 考虑快速开发,报错驱动