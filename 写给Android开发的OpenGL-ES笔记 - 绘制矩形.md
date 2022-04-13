## 写给Android开发的OpenGL-ES笔记 - 绘制矩形

上一篇文章我们绘制了一个三角形，更多的侧重是一个代码的实现。

这篇我们通过绘制一个矩形，来详细说明一下绘制流程。


### 基本概念

#### 基本图元

**OpenGL ES 3.0**可以绘制以下图元

1. 点
2. 线
3. 三角形

**三角形**是我们要介绍的重点**图元** 。**OpenGL ES**支持**三角形图元**包括

1. `GL_TRIANGLES`
2. `GL_TRIANGLE_FAN`
3. `GL_TRIANGLE_STRIP`


让我们用一张图来说明不同的类型，如何绘制矩形。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wetQ9kKljTXuKZdSKRf7WnyicMWiaBHJgEFAKzHkMVRaNEojAc0AtHicz8r3FyJ9jXa9zjB9FZXKRZ9Q/0?wx_fmt=png)

**可以看到，不同的类型决定了是否可以复用顶点以及顶点之间的连接策略**

### 缓冲区对象

接下来记住下面是**三个概念**。

	顶点数组对象：Vertex Array Object，VAO
	顶点缓冲对象：Vertex Buffer Object，VBO
	索引缓冲对象：Element Buffer Object，EBO或Index Buffer Object，IBO
	
同样用一张图来说明问题。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdb2gibo5VqMhbFb1gSDdI06icKEx2XjaKBPOCHteOMLN7R8oPworgxCuIeddiaAV42EKmElsmLibvLWw/0?wx_fmt=png)	

在**OpenGLES2.0** 编程中，用于绘制的顶点数组数据首先保存在 **CPU 内存**，在调用 `glDrawArrays` 或者 `glDrawElements` 等进行绘制时，需要将顶点数组数据从**CPU 内存拷贝到显存**。

但是很多时候我们没必要每次绘制的时候都去进行内存拷贝，如果可以在显存中缓存这些数据，就可以在很大程度上降低内存拷贝带来的开销。

**OpenGLES3.0 VBO 和 EBO 的出现就是为了解决这个问题**

那么问题来了**VAO**是干什么的呢???

	OpenGL的核心模式要求我们使用VAO，所以它知道该如何处理我们的
	顶点输入。如果我们绑定VAO失败，OpenGL会拒绝绘制任何东西。
	
网上好多说没有**VAO**也可以绘制，并且附上了没有使用**VAO**的代码。这个就是只看了表象的结论。

实际上，**OpenGL ES 3.0**中总是有一个活动的**VAO**，所以没有显示的创建**VAO**的都是在这个默认的**VAO**上进行操作。

**VAO是一个状态集，标记的是当前GL_ARRAY_BUFFER Target的数据状态**

每个**VAO**是一个**有限数组**（目前貌似opengl支持最大值16）

这里盗用网上一张大神的图片

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wcYSzr0bMcXjjCOiauDXUx00Cvty73Uqiar5j4LUszMIesd7ibc8uoHd4kO3ibZaXLtOibbjHPGLo4WTuw/0?wx_fmt=jpeg)

**VBO**就是管理了**显存的一块区域**，具体里面的数据是干什么的是描述位置的，还是描述颜色的他不知道。

这时**VAO**就来了，他描述了**VBO**管理的显存的数据的状态。

### 绑定

关于**绑定**的理解不知道是否有新手小伙伴曾经和我有过一样的疑问，我都没有指名要和谁绑定，他怎么生效的呢？？？。

	绑定，就是指定你当前要操作的对象。（opengl 只有一个当前对象，把
	它想象成 current object，你的后续操作，都是在对这个 object 进行。
	你可以拥有很多个 object，然后用 bind 来指定 current object，所以
	你后面的操作，就不需要点名你在操作谁了）
	
好了理论知识暂时就介绍到这里，下面我们开始代码实战。

### 绘制矩形

#### 定义顶点数组

```
val vexCoords = floatArrayOf(
    -0.5f, -0.5f,
    0.5f, -0.5f,
    -0.5f, 0.5f,
    0.5f, 0.5f
)
val verBuffer by lazy {
    // Must use a native order direct Buffer
    ByteBuffer.allocateDirect(vexCoords.size * 4)
        .order(ByteOrder.nativeOrder())
        .asFloatBuffer().also { buffer ->
            buffer.put(vexCoords)
            buffer.position(0)
        }
}
```

#### 定义顶点索引数组

```
val index = intArrayOf(
    0, 1, 2, 1, 2, 3
)
val indexBuffer by lazy {
    ByteBuffer.allocateDirect(index.size * 4)
        .order(ByteOrder.nativeOrder())
        .asIntBuffer().apply {
            put(index)
            position(0)
        }
}
```

#### 生成缓存ID

```
val buffers = IntArray(2)

GLES30.glGenBuffers(vbos.size, buffers, 0)
```

#### 新创建的缓冲绑定到`GL_ARRAY_BUFFER`目标上

```
GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, buffers[0])
```

#### 把之前定义的顶点数据复制到缓冲的内存中

```
GLES30.glBufferData(
    GLES30.GL_ARRAY_BUFFER,
    vexCoords.size * 4,
    verBuffer,
    GLES30.GL_STATIC_DRAW
)
```

#### 新创建的缓冲绑定到`GL_ELEMENT_ARRAY_BUFFER`目标上

```
GLES30.glBindBuffer(GLES30.GL_ELEMENT_ARRAY_BUFFER, buffers[1])
```

#### 把之前定义的顶点索引数据复制到缓冲的内存中

```
GLES30.glBufferData(GLES30.GL_ELEMENT_ARRAY_BUFFER, index.size * 4, indexBuffer, GLES30.GL_STATIC_DRAW)
```

#### 连接顶点属性/启用顶点属性

```
GLES30.glVertexAttribPointer(0, 2, GLES30.GL_FLOAT, false, 2 * 4, 0)
GLES30.glEnableVertexAttribArray(0)
```

#### 绘制

```
GLES30.glDrawElements(GLES30.GL_TRIANGLES, 6, GLES30.GL_UNSIGNED_INT, 0)
```

这里我们使用了**索引法**绘制。在三角形的介绍里面我们用的是**顶点法**绘制。起始我们把索引的点都写到顶点数组中，就是顶点法绘制的数组。

绘制类型我们使用了` GLES30.GL_TRIANGLES`，大家可以自行尝试其他的类型，并修改**起始顶点的位置和顶点的顺序**，看实际的效果。不明白就看之前给的图片。

#### 顶点着色器/片段着色器

这部分还是使用的之前绘制三角形的着色器代码。不再啰嗦介绍了。关于加载着色器，连接程序，最后激活程序也都不啰嗦了，请参考完整项目代码。


### 完整项目地址

[https://github.com/flyminiboy/LearnOpenGLES](https://github.com/flyminiboy/LearnOpenGLES)

**关于使用VAO的方式，请直接参考完整项目**

### 结语

这里我们就不介绍不使用**VBO**的绘制方式了。请使用现代化的方式！！！


