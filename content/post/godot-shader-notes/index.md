---
title: "Godot Shader 笔记"
description: Godot Shader 常用公式、内置值、渲染阶段与光照机制速查
date: 2026-09-30T10:00:00+08:00
image: 
math: true
license: 
comments: true
draft: false
categories: ["Godot"]
tags: ["shader", "godot", "技术美术"]
build:
    list: always
---

## GodotShader

### 常规公式

- 点到线的距离
通用线表达式;
`Ax + By + C = 0`;
点到该线距离

$$d=\frac{Ax+By​+C}{\sqrt{A^2 + B^2}}​$$

斜率式且不考虑偏移:
`y - s * x = 0` s为斜率
$$d=\frac{y-sx}{\sqrt{1 + s^2}}​$$

由逆变换可得, 线条偏移仅需计算. / 或者直接偏移uv值后代入 `y = y+offset_y; x = x + offset_x'

$$d=\frac{y-sx}{\sqrt{1 + s^2}} + c​$$


- 随机雪花点
```glsl
float random(vec2 uv) {
	return fract(sin(dot(uv.xy, vec2(12.9821, 78.2332))) * 43785.52344654);
}
```

- 宽高比
```
float radio = dFdy(UV.y) / dFdx(UV.x);
```

- 窗口宽高比
```
float s_radio = SCREEN_PIXEL_SIZE.y / SCREEN_PIXEL_SIZE.x;
```

- 偏移 + Mod移动角起点
```glsl
vec2 uv = 2. * UV - 1.;
float angle = atan(uv.y, uv.x) + PI / 2.;
angle = mod(angle, TAU);
float mask = step(angle, progress * TAU);
```
如果只是希望转90°获取值, 那么交换 x, y 轴就行了. 
```
float main_dial_mask = (atan(uv.x, uv.y) + PI) / TAU;
```
以及更多操作都可以从这两个方向
	- 修改结果值.
	- 修改传入参数.

- 剔除不在范围内的UV导致的重复

```
vec2 s = step(vec2(0.), UV) * step(UV, vec2(1.));
COLOR.a *= s.x * s.y;
```

- HSV - HUE

不过更推荐网上去找色调生成算法网站, 搭配自己的色调
```glsl
vec3 hsv2rgb(float hue) {
	vec3 k = vec3(1. , 2./ 3., 1./3.);
	vec3 p = abs(fract(vec3(hue) + k) * 6. -3.);
	return clamp(p - vec3(1.), 0, 1.);
}
```
### Func

- smoothstep: 平滑阶跃, edge 0 - edge 1. edge1之后为1, edge0之前为0

- round: 四舍五入
- distance: 计算距离

- mix: c1, c2, p  return `p * c2  + (1 - p) * c1` 

- fract: 取小数点后
- mod:取模, 不同于 `%` , 其也可以接受float参数

- dFdy: y方向上求导/数值差分: `dFdy(x) ≈ x(u, v+1) - x(u, v)`
	- dFdy(UV.y) =  UV.y(pixel_y) - UV.y(pixel_y  - 1)
	- 当然 dFdy(UV.x) 就是0了,y方向上 UV.x不会变化
- dFdx: x方向上求导

> **`dFdy(UV.y)` 能起作用，并不是因为它会“求导”这个数字，而是因为 GPU 会在同一时刻，用同一个着色器代码去计算屏幕上相邻的多个像素，然后直接把它们的结果相减。**
>  利用好UV差分可以计算Item的拉伸比率

- atan: 反正切
	- 除了直接传入float值, 也支持传入y, x两个值.
	- 两个值传入时, 会依靠y,x正负符号判断所在象限. 
	- 类似与直接在圆上选点, 此时atan能区分四个象限.对于3, 4象限值则是负角度值
		- 3象限 为  atan(y/x) - pi
		- 2象限 为 atan(y/x) + pi
		- 不过更容易理解的方法就是不超过pi的从右x轴开始的正负角取值

### 内置值
- TEXTURE_PIXEL_SIZE: 1/贴图尺寸(像素个数) , 即一个贴图像素对应uv的值.  `vec2(1.0 / texture_width, 1.0 / texture_height)`
- SCREEN_PIXEL_SIZE: `vec2(1.0 / viewport_width, 1.0 / viewport_height)` 视口像素尺寸, 动态随窗口拉伸变化
- **VERTEX**: 
	- vertex阶段时, 其为顶点局部坐标(中心为pivot), 单位为像素.
		- 你可以用顶点阶段计算出screen_pos: `CANVAS_MATRIX * MODEL_MATRIX * vec4(VERTEX, 0, 1.)`
	- fragment阶段时, 其为每个像素的屏幕坐标 screen_pos, 单位为像素, 使用 SCREEN_PIXEL_SIZE可归一化.
		- SCREEN_UV = VERTEX * SCREEN_PIXEL_SIZE
		- FRAGCOORD.xy 也是 screen_pos
- **COLOR**:
	- vetex 阶段时为顶点叠加色, 等同于 Modulate / self Modulate. 使用COLOR时 这两个都会失效
	- fragment: 像素颜色
- UV:
	- 两者一致, texture采样位置, 左上角起点, 归一化.
	- Polygon2d中的轮廓点, 在贴图外时会其点的UV值其实实在 0~1范围外的

#### 顶点内置
  
| 内置                             | 描述                                                                                                                                                                                                                                                                                                          |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| in mat4 **MODEL_MATRIX**       | **从局部空间到世界空间的转换矩阵。**世界空间，其实就是你在编辑器里平时操作时用到的那些坐标。                                                                                                                                                                                                                                                            |
| in mat4 **CANVAS_MATRIX**      | **从世界空间到画布空间的转换矩阵**。在画布空间中，屏幕的左上角是坐标原点 (0, 0)，坐标范围从 `(0.0, 0.0)` 一直延伸到视口的大小。                                                                                                                                                                                                                                |
| in mat4 **SCREEN_MATRIX**      | 从画布空间（Canvas space）到裁剪空间（Clip space）的变换。在裁剪空间中，坐标的取值范围是从 `(-1.0, -1.0)` 到 `(1.0, 1.0).`                                                                                                                                                                                                                     |
| in int **INSTANCE_ID**         | 实例化的实例 ID。                                                                                                                                                                                                                                                                                                  |
| in vec4 **INSTANCE_CUSTOM**    | 实例自定义数据.                                                                                                                                                                                                                                                                                                    |
| in bool **AT_LIGHT_PASS**      | 始终为 `false`。                                                                                                                                                                                                                                                                                                |
| in vec2 **TEXTURE_PIXEL_SIZE** | 默认 2D 纹理的归一化像素尺寸。对于一个纹理尺寸为 64x32 像素的 Sprite2D 来说，`TEXTURE_PIXEL_SIZE` = `vec2(1.0/64.0, 1.0/32.0)`。                                                                                                                                                                                                         |
| inout vec2 **VERTEX**          | 顶点位置，使用局部空间。                                                                                                                                                                                                                                                                                                |
| in int **VERTEX_ID**           | 顶点缓冲区中当前顶点的索引。                                                                                                                                                                                                                                                                                              |
| inout vec2 **UV**              | 归一化的纹理坐标。范围从 `0.0` 到 `1.0`。                                                                                                                                                                                                                                                                                 |
| inout vec4 **COLOR**           | 来自顶点图元的颜色，乘以 CanvasItem 的 [modulate](https://docs.godotengine.org/zh-cn/4.x/classes/class_canvasitem.html#class-canvasitem-property-modulate) （调制色），再乘以 CanvasItem 的 [self_modulate](https://docs.godotengine.org/zh-cn/4.x/classes/class_canvasitem.html#class-canvasitem-property-self-modulate) （自身调制色）。 |
| inout float **POINT_SIZE**     | 点绘图的点大小.                                                                                                                                                                                                                                                                                                    |
| in vec4 **CUSTOM0**            | 来自顶点图元的自定义值。                                                                                                                                                                                                                                                                                                |
| in vec4 **CUSTOM1**            | 来自顶点图元的自定义值。                                                                                                                                                                                                                                                                                                |

#### 光源内置

LIGHT_DIRECTION: 原点为光源中心点, 方向为从原点到像素的方向向量
LIGHT_POSITION: 光源中心在屏幕空间的像素位置
LIGHT_VERTEX: 像素在屏幕空间的像素位置
SPECULAR_SHININESS: 高光贴图采样 vec4
### 魔法数字
- uniform vec3 gray_weights = vec3(.299, .587, .114); : 灰度计算权重

## 机制解释

### 法线贴图光照:
```
float CNdotL = max(0.0, dot(NORMAL, LIGHT_DIRECTION));
LIGHT = vec4(COLOR.rgb * LIGHT_COLOR.rgb * LIGHT_ENERGY * CNdotL, 1.);

```

### +反射贴图

```
float CNdotL = max(0.0, dot(NORMAL, LIGHT_DIRECTION));
LIGHT = vec4(COLOR.rgb * LIGHT_COLOR.rgb * LIGHT_ENERGY * CNdotL * SPECULAR_SHININESS.rgb, 1.);

```
### 光源:

和其他引擎一样, 物体像素受到几个光源阴影, light就会执行几次

```
void light() {
	COLOR; // From Fragment Color ouput
	
	// 默认光源计算式
	LIGHT = vec4(COLOR.rgb * LIGHT_COLOR.rgb * LIGHT_ENERGY, 1.);
	// Called for every pixel for every light affecting the CanvasItem.
	// Uncomment to replace the default light processing function with this one.
}
```

LightOccluder2D 节点遮挡光源, 形成引用 (Light自身开启shadow)
### 渲染内容捕获

sampler2D: 
- hint_screen_texture 说明会捕获此前渲染的屏幕内容
- filter_linear_mipmap: 使用线性过滤的纹理映射 - > 可用于实现模糊, 
	- 此时需要使用 textureLod 采样

backBufferCopy 节点:
> 这种节点能够将屏幕中的某个区域复制到缓冲中，方便从着色器代码中访问。
> 用于后台缓冲当前显示屏幕的节点。会根据 copy_mode 对 BackBufferCopy 节点中定义的区域所覆盖的屏幕内容或整个屏幕进行缓冲。可以在着色器脚本中使用屏幕纹理来访问（即带有 hint_screen_texture 的 uniform 采样器）。

当存在节点时, 其会将此前渲染内容缓存/复制, 此时其后的节点材质中, hint_screeen_texture获取的纹理即为其缓存的纹理. (viewport 模式), 
当然, 由于是捕获渲染内容, 节点的可见性也会影响获取的纹理


```glsl
uniform sampler2D screen_texture: hint_screen_texture;
```
### Vertex -> Fragment 阶段
varying 都会在从vertex 到 fragment阶段中被插值

>  
>  哦对了, godot shader中 通过varying 声明自定义传递数据
>  
```
shader_type canvas_item;
varying vec2 vertex_varying;

void vertex() {
	VERTEX.x += sin(VERTEX.y) * 64.;
	vertex_varying = VERTEX;
	// Called for every vertex the material is visible on.
}

void fragment() {
	COLOR.rgb = vec3(vertex_varying / 64., 0.);
}

```
