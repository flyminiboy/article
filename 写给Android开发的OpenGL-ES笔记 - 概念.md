## 写给Android开发的OpenGL-ES笔记 - 概念

在编写具体的代码之前我们首先要搞明白一些基本的**概念**，这样在后面写代码的时候就会少很多的疑问。这篇内容不涉及代码编写，但是**很重要**。

### OpenGL 是什么

**OpenGL规范** 描述了绘制2D和3D图形的抽象API

**OpenGL的API** 定义了若干可被客户端程序调用的**函数**，以及一些具名**整型常量**。

**OpenGL** 不仅语言无关，而且平台无关。规范只字未提获得和管理**OpenGL上下文**相关的内容，而是将这些作为细节交给底层的**窗口系统**。(这部分知识很重要，后面介绍 **EGL** 就是基于这部分知识的。)

**OpenGL** 有许多语言绑定，**Android**提供的**Java**和**C**绑定。

### OpenGL ES 是什么

**OpenGL ES** 是 **OpenGL** 的一个子集，是以支持手持和嵌入式设备为目标的高级3D图形应用程序编程接口（API）

到目前为止，有四种规范。

1. OpenGL ES 1.X 采用固定功能管线
2. OpenGL ES 2.0 采用了可编程图形管线
3. OpenGL ES 3.0 采用了可编程图形管线，加入许多2.0不支持的功能


**可编程图形管线**开发者可以自己编写图形管线中的 `顶点着色器` 和 `片段着色器` 两个阶段的代码

**Android 4.3 之后开始支持 OpenGL ES 3.0**

我们后续代码都是基于 **OpenGL ES 3.0** 进行，所以我们的介绍也都是基于 **OpenGL ES 3.0规范**。

**OpenGL ES 3.0规范** 包括俩个部分

1. OpenGL ES 3.0 API规范
2. OpenGL ES 着色语言 3.0 规范（OpenGL ES SL）后面会介绍

**图形渲染管线**

这个我们直接看 **LearnOpenGL** 网站 对这个概念的解释
> 图形渲染管线（Graphics Pipeline，大多译为管线，实际上指的是一堆原始图形数据途经一个输送管道，期间经过各种变化处理最终出现在屏幕的过程）管理的

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfeDs91UXxC7yz5J4qELVxm0TsVGhcu8iaGM75BuqgNejVAN2PYQpxW5DickGPibOtW7L2CNEE22ibVYQ/0?wx_fmt=png)

这幅图当中，**顶点缓冲区/数组对象，顶点着色器，纹理坐标，片段着色器**，是我们平时开发中可以直接接触到的一些东西，我们在后面也会做一些相应的介绍。(这部分知识很重要，后面介绍 **EGL** 就是基于这部分知识的。)

**OpenGL ES 绘制前提**

1. 上下文环境 - 储存OpenGL ES状态
2. 绘制表面 - 绘制图元的表面

**在 OpenGL 的设计中，OpenGL 不负责窗口的管理，和上下文环境的管理**



### GLSL 是什么

**GLSL** 是 OpenGL ES Shading Language 的缩写。

在介绍 **OpenGL ES** 的时候，我们提到了从 **OpenGL ES 2.0规范** 开始采用了 **可编程图形管线**。开发者可以自己使用` OpenGL ES Shading Language（着色语言）` 来编写 `顶点着色器` 和 `片段着色器` 两个阶段的代码。

关于 **OpenGL ES** 与 **OpenGL ES Shading Language** 的对应关系

| OpenGL ES Version |	GLSL Version |
|  ----  | ----  |
|1.0	|--|
|1.1	|--|
|2.0	|100|
|3.0	|300|
|3.1	|310|
|3.2	|320|

#### 着色器

着色器是使用一种 **GLSL** 的类C语言写成的

着色器的开头总是要`声明版本`，接着是`输入变量`和`输出变量`、`uniform`和`main函数`

每个着色器的入口点都是`main函数`

#### 输入和输出

我们希望每个**着色器**都有**输入和输出**，这样才能进行数据**交流和传递**，这和我们前面的 **图形渲染管线** 就联系上了，当然如果大家还是有一点点的疑惑可以类比我们熟悉的Android开发，这个其实就是类似我们在Android开发的时候，Google为我们提供的 **gradle transform API** 一样，有**输入和输出**，上一个 **transform** 的**输出**作为下一个 **transform** 的**输入**。

**GLSL** 定义了 `in` 和 `out` 关键字专门来实现这个目的。

**顶点着色器**的输入特殊在，它从**顶点数据**中直接接收输入。

`layout (location = 0)` 布局标识符，可以帮助我们快速的把**顶点着色器**链接到**顶点数据**
> 你也可以忽略layout (location = 0)标识符，通过在OpenGL代码中使用glGetAttribLocation查询属性位置值(Location)，但是我更喜欢在着色器中设置它们，这样会更容易理解而且节省你（和OpenGL）的工作量

#### Uniform

**Uniform** 是一种从CPU中的应用向GPU中的着色器发送数据的方式，但uniform和顶点属性有些不同。首先，uniform是全局的(Global)；第二，无论你把uniform值设置成什么，uniform会一直保存它们的数据，直到它们被重置或更新。

**uniform对于设置一个在渲染迭代中会改变的属性是一个非常有用的工具，它也是一个在程序和着色器间数据交互的很好工具**

下面我们看看俩个关于着色器的小示例

#### 顶点着色器

```
#version 300 es // 声明版本

layout (location = 0) in vec4 vPosition; // 定义输入变量

void main() { // 入口main函数
    gl_Position = vPosition; // gl_Position 内建变量，是一个vec4类型
}
```

#### 片段着色器/片元着色器

```
#version 300 es
precision mediump float;

out vec4 outColor; // 定义输出变量

void main() {
    outColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);
}
```

#### 补充说明 关于 OpenGL ES 3.0 的小改动

1. OpenGL ES 3.0中将2.0的attribute改成了in，顶点着色器的varying改成out，片段着色器的varying改成了in，也就是说顶点着色器的输出就是片段着色器的输入，另外uniform跟2.0用法一样
2. GLSL 3.0 将`gl_FragColor`和`gl_FragData`内置变量取消掉了，需要自己定义out变量作为片段着色器的输出颜色，如 `out vec4 fragColor`
3. OpenGL ES 3.0的shader中没有`texture2D`和`texture3D`等了，全部使用`texture`替换

### 纹理

**纹理** 是一个2D图片（甚至也有1D和3D的纹理），它可以用来添加物体的细节

我们可以通过这样一个场景来想象**纹理**这个东西，比如我们要装饰电视墙（**可绘区域**），我们现在有俩个选择，一是直接画上去；二是我们可以买壁纸贴上去，那么壁纸其实就是我们的纹理。

我们通过一个小示例来说明纹理

为了能够把**纹理映射(Map)到三角形**上，我们需要指定三角形的每个顶点各自对应纹理的哪个部分。这样**每个顶点就会关联着一个纹理坐标(Texture Coordinate)**，用来标明从该纹理图像的哪个部分采样（译注：采集片段颜色）。之后在图形的其它片段上进行片段插值(Fragment Interpolation)。

**纹理坐标在x和y轴上，范围为0到1之间**（注意我们使用的是2D纹理图像）。使用纹理坐标获取纹理颜色叫做**采样(Sampling)**。纹理坐标起始于(0, 0)，也就是纹理图片的左下角，终始于(1, 1)，即纹理图片的右上角。


### 坐标问题

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wfeDs91UXxC7yz5J4qELVxmKDSKLpnFYTceJ6jcn5wjk1l09tafjFFibjBo6r5bfuz3F3JiciaktbStg/0?wx_fmt=jpeg)

坐标这个我们先把这个图记下来，后续我们还会进一步介绍。

### EGL

#### EGL 是什么

**EGL 是一个通过操作系统创建和访问窗口的库提供了渲染API（OpenGL ES）和原生窗口系统之间的接口**

#### EGL 作用

* 与设备的原生窗口系统通信；
* 查询绘图表面的可用类型和配置；
* 创建绘图表面；
* 在OpenGL ES 和其他图形渲染API之间同步渲染；
* 管理纹理贴图等渲染资源。

#### 为什么要使用 EGL 

还记得我们在介绍 **OpenGL** 的时候，说的一段内容了吗

> **OpenGL** 不仅语言无关，而且平台无关。规范只字未提获得和管理**OpenGL上下文**相关的内容，而是将这些作为细节交给底层的**窗口系统**。

我们再看Android官方的一段解释

> OpenGL ES 定义了一个渲染图形的 API，但没有定义窗口系统。为了让 GLES 能够适合各种平台，GLES 将与知道如何通过操作系统创建和访问窗口的库结合使用。用于 Android 的库称为 EGL。如果要绘制纹理多边形，应使用 GLES 调用；如果要在屏幕上进行渲染，应使用 EGL 调用。


**所以说，EGL 是 OpenGL ES 和具体的平台之间的中间层，用来做适配的。Android EGL 提供了本地平台对 OpenGL ES 的实现**


好了关于一些必须了解的基本概念我们就介绍到这里了。下一篇开始我们要开始正式的代码编写了。
	
