## 写给Android开发的OpenGL-ES笔记 - 绘制矩形

本来没有这篇，但是经过了这阶段的学习，觉得有必要再来这么一篇。主要还是为了让小伙伴们**彻底搞明白**绘制一块的东西。也许网上其他人的写文章的人懂，但是它们没有写出来，也许它们也根本不明白。

当然我这篇文章也不一定对，但是我只是希望从使用的角度上，让大家做到心里有数，一些原理理论知识我也没有理解透彻。请不要喷，互相学习。

还是一样先开始我们的**理论知识**补充。

### 基本概念

#### 显卡
主机里的数据要显示在屏幕上就需要**显卡**。具体来说，显卡接在电脑主板上，**它将电脑的数字信号转换成模拟信号让显示器显示出来**

#### GPU
**GPU（Graphic Processing Unit）全称图形处理单元**。GPU是显卡上的一块芯片，就像CPU是主板上的一块芯片

#### 显存
**显存（Video Memory），也被叫做帧缓存**，它的作用是用来存储显卡芯片处理过或者即将提取的渲染数据。



#### 基本图元
**OpenGL**

首先在**OpenGLES2.0** 编程中，用于绘制的顶点数组数据首先保存在 **CPU 内存**，在调用 `glDrawArrays` 或者 `glDrawElements` 等进行绘制时，需要将顶点数组数据从**CPU 内存拷贝到显存**。

但是很多时候我们没必要每次绘制的时候都去进行内存拷贝，如果可以在显存中缓存这些数据，就可以在很大程度上降低内存拷贝带来的开销。

**OpenGLES3.0 VBO 和 EBO 的出现就是为了解决这个问题**

接下来记住下面是**三个概念**。补充知识就是围绕它们展开的。

	顶点数组对象：Vertex Array Object，VAO
	顶点缓冲对象：Vertex Buffer Object，VBO
	索引缓冲对象：Element Buffer Object，EBO或Index Buffer Object，IBO
	
![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wc0DfbKkQZ5wbic9TaMpNu2jtyC8tlhv5ia7PVxC9JuUVsaybBhIgxFqhg8DVIqnrfiapoVv1uGT50AQ/0?wx_fmt=png)
	
当我们定义好**顶点数据**以后，我们会把它作为输入发送给图形渲染管线的第一个处理阶段：**顶点着色器**

它（顶点着色器）会在**GPU上创建内存**用于储存我们的顶点数据

我们通过**顶点缓冲对象(Vertex Buffer Objects, VBO)**管理这个内存，它会在**GPU内存**（通常被称为显存）中储存大量顶点

	注意！！！，VBO 是管理顶点着色器创建的内存，是管理！！！

* 首先使用`glGenBuffers`函数生成一个**缓冲ID**

```
val vbos = IntArray(1)

GLES30.glGenBuffers(1, vbos, 0)
```

**OpenGL**有很多缓冲对象类型，**顶点缓冲对象**的缓冲类型是`GL_ARRAY_BUFFER`

* 新创建的缓冲绑定到`GL_ARRAY_BUFFER`目标上

```
GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, vbos[0])
```

* 把之前定义的**顶点数据**复制到缓冲的内存中

```
GLES30.glBufferData(GLES30.GL_ARRAY_BUFFER, vexCoords.size * 4, verBuffer, GLES30.GL_STATIC_DRAW)
```

`glBufferData`是一个专门用来把**用户定义的数据复制**到当前**绑定缓冲**的函数（**CPU内存数据复制到GPU内存-显卡**）

**第四个参数**指定了我们希望显卡如何管理给定的数据。它有三种形式：

* GL\_STATIC\_DRAW ：数据不会或几乎不会改变。
* GL\_DYNAMIC\_DRAW：数据会被改变很多。
* GL\_STREAM\_DRAW ：数据每次绘制时都会改变。

		每个顶点属性从一个VBO管理的内存中获得它的数据，
		而具体是从哪个VBO（程序中可以有多个VBO）获取
		则是通过在调用glVertexAttribPointer时绑定到GL_ARRAY_BUFFER的VBO决定的。
		
这个地方具体从哪个VBO获取不知道大家看明白说明没有啊，不明白，小弟就给来一个通俗的表达，就是在调用`glVertexAttribPointer `时候，距离最近的一次绑定。所以一般我们在使用**VBO**的时候下面这段代码都是一起出现。

```
GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, vbos[0])
GLES30.glBufferData(GLES30.GL_ARRAY_BUFFER, vexCoords1.size * 4, verBuffer1, GLES30.GL_STATIC_DRAW)
GLES30.glVertexAttribPointer(0, 2, GLES30.GL_FLOAT, false, 2 * 4, 0)
GLES30.glEnableVertexAttribArray(0)
```
	
**顶点数组对象(Vertex Array Object, VAO)**可以像顶点缓冲对象那样被

**OpenGLES3.0 VBO 和 EBO 的出现就是为了解决这个问题**。 

**OpenGLES3.0** 支持两类缓冲区对象：

	1. 顶点数组缓冲区对象 VBO
	2. 图元索引缓冲区对象 EBO/IBO


先看看我们之前是如何做得**链接顶点属性**

`GLES30.glVertexAttribPointer(0, 2, GLES30.GL_FLOAT, false, 2 * 4, verBuffer)`

**注意最后一个参数我们是传递了一个**`FloatBuffer`。



