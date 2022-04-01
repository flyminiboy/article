## 写给Android开发的OpenGL-ES笔记 - 绘制三角形


这篇文章的最终目的是要在屏幕上绘制一个三角形出来。

### 理论基础

基于我的一贯风格。我们还是先说几个概念，先把概念默默的在心里面有个印象，不要后面在看见的时候，无数个草泥马狂奔。
	
盗图一张，下面这幅图说明了**图形渲染管线**的每个阶段的抽象展示。**要注意蓝色部分代表的是我们可以注入自定义的着色器的部分**
	
![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdoIw3vWX7VGAbxlBibEdgiaCcn5EiconkxMfsyd9gjj06FCiamfmibH3uaa2nicprE1kDUErTr4Yp6RnvQ/0?wx_fmt=png)

在正式写代码之前，我们先来描述一下我们的工作流程。

首先，我们以**数组**的形式传递3个3D/2D坐标作为图形渲染管线的**输入**，用来表示一个三角形，这个数组叫做**顶点数据(Vertex Data)**

然后，**顶点着色器(Vertex Shader)**，它把一个单独的顶点作为输入，内部主要是做些坐标的转换。

再经过 **图元装配(Primitive Assembly)** ，**几何着色器(Geometry Shader)** ，**光栅化阶段(Rasterization Stage)** 最终生成供**片段着色器(Fragment Shader)**使用的**片段(Fragment)（OpenGL中的一个片段是OpenGL渲染一个像素所需的所有数据。）**

上面三步操作我们无法手动控制，所以不再做更多的介绍。

然后**片段着色器** 计算一个像素的最终颜色。

最后最终的对象将会被传到最后一个阶段，我们叫做**Alpha测试和混合(Blending)**阶段，这一步我们也无法手动控制，所以不再做更多介绍。

盗图一张

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdoIw3vWX7VGAbxlBibEdgiaCjxjmsG9QEUL6H7FNQj17eX2MP7ftHgXXXwdPLrwREwC4jLKR3Y4gvw/0?wx_fmt=png)

左边是OpenGL坐标，右边是Android屏幕的映射。

一旦我们的**顶点坐标**已经在**顶点着色器**中处理过，它们就应该是**标准化设备坐标**了，标准化设备坐标是一个**x、y和z值在-1.0到1.0**的一小段空间。（从现在开始我们依然会说3D坐标，但是我们实际只做了一个2D的图形，所以我们默认忽略z轴）

### 编写代码

好了下面我们正式开始我们这一系列文章的代码之旅了。

其实**OpenGL ES**的大部分代码都是一些模板代码是固定的格式。

所以我们有时候要先有**don't ask why，just do it**的觉悟。

我们现在这个版本是基于 `GLSurfaceView` 和`GLSurfaceView.Renderer`创建视图容器。

`GLSurfaceView` 提供了用于管理 **EGL 上下文**、线程间通信以及与 Activity 生命周期的交互的辅助程序类，所以就目前而言，他可以帮助我们简化我们的开发流程，降低开发难度。

#### 声明 OpenGL 要求

对于 OpenGL ES 3.0：

```
<!-- Tell the system this app requires OpenGL ES 3.0. -->
<uses-feature android:glEsVersion="0x00030000" android:required="true" />
    
```

首先我们要定义好**顶点着色器**和**片段着色器**。有俩种方式，

1. 直接将顶点着色器的源代码硬编码在代码文件顶部的字符串中。
2. main/res/raw 下面新建俩个.glsl 文件。

这里我们采用第二种方式

**triangle\_vertex\_shader.glsl** 顶点着色器

```
#version 300 es
layout (location = 0) in vec4 vPosition;

void main() {
    gl_Position = vPosition; // gl_Position 内建变量，是一个vec4类型
}
```

**triangle\_fragment\_shader.glsl** 片段着色器

```
#version 300 es
precision mediump float;

out vec4 outColor;

void main() {
    outColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);
}
```

自定义`MyGLSurfaceView`（非必须）。设置参数

```
class MyGLSurfaceView(context: Context, attrs: AttributeSet?=null) : GLSurfaceView(
    context, attrs
) {

    init {
        setEGLContextClientVersion(3)
        setRenderer(MyGLRender(this))
        // 设置刷新模式
        renderMode = RENDERMODE_WHEN_DIRTY
    }

}
```

自定义`MyGLRender `实现`GLSurfaceView.Renderer`接口

在`onSurfaceCreated`做一些初始化操作。

```
        val vertexShaderSource = loadShaderSource(context, R.raw.triangle_vertex_shader)
        val fragmentShaderSource = loadShaderSource(context, R.raw.triangle_fragment_shader)

        val vertexShader = loadShader(GLES30.GL_VERTEX_SHADER, vertexShaderSource)
        val fragmentShader = loadShader(GLES30.GL_FRAGMENT_SHADER, fragmentShaderSource)

        program = createAndLinkProgrm(context, vertexShader, fragmentShader)

        // VAO
//        GLES30.glGenVertexArrays(1, vaosBuffer)
        // VBO
//        GLES30.glGenBuffers(1, vbosBuffer)
        // 要想使用VAO，要做的只是使用glBindVertexArray绑定VAO
//        GLES30.glBindVertexArray(vaosBuffer[0])
        // 复制顶点数组到缓冲中供OpenGL使用
//        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, vbosBuffer[0])
//        GLES30.glBufferData(GLES30.GL_ARRAY_BUFFER, vexCoords.size * 4, verBuffer, GLES30.GL_STATIC_DRAW)

        // 链接顶点属性
        // 第一个参数指定我们要配置的顶点属性。
        // 还记得我们在顶点着色器中使用layout(location = 0)定义了position顶点属性的位置值(Location)吗？
        // 它可以把顶点属性的位置值设置为0。因为我们希望把数据传递到这一个顶点属性中，所以这里我们传入0

        // 第二个参数指定顶点属性的大小。这里我们在定义顶点数组是定义了一个2D的图形，只有x,y俩个维度，所以大小是2
        // 第三个参数指定数据的类型，这里是GL_FLOAT(GLSL中vec*都是由浮点数值组成的)

        // 是否希望数据被标准化(Normalize)。如果我们设置为GL_TRUE，所有数据都会被映射到0（对于有符号型signed数据是-1）到1之间。
        // 我们把它设置为GL_FALSE

        // 步长(Stride)，它告诉我们在连续的顶点属性组之间的间隔 由于下个组位置数据在2个float之后 我们把步长设置为3 * sizeof(float)4
        // 表示位置数据在缓冲中起始位置的偏移量,由于位置数据在数组的开头，所以这里是0.在本示例中我们指定以了位置数据

        //
        GLES30.glVertexAttribPointer(0, 2, GLES30.GL_FLOAT, false, 2 * 4, verBuffer)
        // 使用VBO + VAO 优化
//        GLES30.glVertexAttribPointer(0, 2, GLES30.GL_FLOAT, false, 2 * 4, 0)
        // 现在我们已经定义了OpenGL该如何解释顶点数据，我们现在应该使用glEnableVertexAttribArray，以顶点属性位置值作为参数，启用顶点属性
        GLES30.glEnableVertexAttribArray(0)

        //8. 解绑VBO
//        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER,0)
//        GLES30.glBindVertexArray(0)
```

在 `onSurfaceChanged` 设置**视口维度**

```
    override fun onSurfaceChanged(gl: GL10?, width: Int, height: Int) {
        // 设置窗口的维度
        // glViewport函数前两个参数控制窗口左下角的位置。第三个和第四个参数控制渲染窗口的宽度和高度（像素）
        // OpenGL幕后使用glViewport中定义的位置和宽高进行2D坐标的转换，将OpenGL中的位置坐标转换为你的屏幕坐标
        GLES30.glViewport(0, 0, width, height)
    }
```

在 `onDrawFrame` 开始我们的绘制

```
    override fun onDrawFrame(gl: GL10?) {

        program?.let {

            // 使用一个自定义的颜色清空屏幕
            // 在每个新的渲染迭代开始的时候我们总是希望清屏，否则我们仍能看见上一次迭代的渲染结果

            // 设置清空屏幕所用的颜色 当调用glClear函数，清除颜色缓冲之后，整个颜色缓冲都会被填充为glClearColor里所设置的颜色
            GLES30.glClearColor(0.2f, 0.3f, 0.3f, 1.0f)
            // 我们可以通过调用glClear函数来清空屏幕的颜色缓冲,glClear它接受一个缓冲位(Buffer Bit)来指定要清空的缓冲,这里我们只关心颜色
            GLES30.glClear(GLES30.GL_COLOR_BUFFER_BIT)

            // 激活程序对象
            // 在glUseProgram函数调用之后，每个着色器调用和渲染调用都会使用这个程序对象（也就是之前写的着色器)了
            GLES30.glUseProgram(it)

//            GLES30.glBindVertexArray(vaosBuffer[0])

            // 使用当前激活的着色器，之前定义的顶点属性配置，和VBO的顶点数据（通过VAO间接绑定）来绘制图元。
            // 第一个参数是我们打算绘制的OpenGL图元的类型
            // 指定了顶点数组的起始索引
            // 指定我们打算绘制多少个顶点


            GLES30.glDrawArrays(GLES30.GL_TRIANGLES, 0, 3)

        }


    }
```


### 分析

首先为了能够让**OpenGL**使用着色器，我们必须在运行时动态编译它的源代码

```
fun loadShader(shaderType: Int, shaderSource: String) =

    // 根据着色器类型创建着色器对象
    GLES30.glCreateShader(shaderType).also { shader ->
        // 将着色器源码附加到着色器上
        GLES30.glShaderSource(shader, shaderSource)
        GLES30.glCompileShader(shader)
        // 检测是否编译成功
        val compiled = intArrayOf(1)
        GLES30.glGetShaderiv(shader, GLES30.GL_COMPILE_STATUS, compiled, 0)
        if (compiled[0] == GLES30.GL_FALSE) {
            val error = GLES30.glGetShaderInfoLog(shader)
            logE("glCompileShader info [ $error ]")
            return GLES30.GL_FALSE
        }
    }
```

### 着色器程序

**着色器程序对象(Shader Program Object)**是多个着色器合并之后并最终链接完成的版本

要使用刚才编译的着色器我们必须把它们链接(Link)为一个着色器程序对象，然后在渲染对象的时候激活这个着色器程序。

```
fun createAndLinkProgrm(context: Context, vertexShader: Int, fragmentShader: Int) =
    // 构建着色器程序对象
    GLES30.glCreateProgram().also { program ->

        // 将着色器附加到着色器程序
        GLES30.glAttachShader(program, vertexShader)
        GLES30.glAttachShader(program, fragmentShader)
        // 链接程序
        GLES30.glLinkProgram(program)
        // 检测链接状态
        val linked = intArrayOf(1)
        GLES30.glGetProgramiv(program, GLES30.GL_LINK_STATUS, linked, 0)
        if (linked[0] == GLES30.GL_FALSE) {
            val error = GLES30.glGetProgramInfoLog(program)
            logE("glCompileShader info [ $error ]")
            return GLES30.GL_FALSE
        }

        // 在把着色器对象链接到程序对象以后，记得删除着色器对象，我们不再需要它们了
        GLES30.glDeleteShader(vertexShader)
        GLES30.glDeleteShader(fragmentShader)
    }
```

### 顶点输入

```
    /**
     * 顶点坐标
     *
     * 定义这样的顶点数据以后，我们会把它作为输入发送给图形渲染管线的第一个处理阶段：顶点着色器
     *
     * 它会在GPU上创建内存用于储存我们的顶点数据
     *
     * 我们通过顶点缓冲对象(Vertex Buffer Objects, VBO)管理这个内存,类型 GL_ARRAY_BUFFER
     */
    val vexCoords = floatArrayOf(
        -0.5f, -0.5f,
        0.5f, -0.5f,
        0f, 0.5f
    )
```

注意这里关于顶点的定义，我们直接采用了2D的标准，只包括`x和y`，并么有声明`z`，这涉及到我们在 **链接顶点属性** 时候的一些参数设置。

### 链接顶点属性

继续盗图一张。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdoIw3vWX7VGAbxlBibEdgiaCHXXy5r3vXniczqeC9TGfjG6I2cJ6Wo9zbORA6dVF19MVpgbtpicIiaPBQ/0?wx_fmt=png)

我们必须手动指定输入数据的哪一个部分对应顶点着色器的哪一个顶点属性，所以，我们必须在渲染前指定OpenGL该如何解释顶点数据

我们的顶点缓冲数据最终会被解析为上图这样子。（注意图里面是以3D坐标说明的，即每个顶点包括`x,y,z`）

```
// 链接顶点属性
// 第一个参数指定我们要配置的顶点属性。
// 还记得我们在顶点着色器中使用layout(location = 0)定义了position顶点属性的位置值(Location)吗？
// 它可以把顶点属性的位置值设置为0。因为我们希望把数据传递到这一个顶点属性中，所以这里我们传入0
// 第二个参数指定顶点属性的大小。这里我们在定义顶点数组是定义了一个2D的图形，只有x,y俩个维度，所以大小是2
// 第三个参数指定数据的类型，这里是GL_FLOAT(GLSL中vec*都是由浮点数值组成的)
// 是否希望数据被标准化(Normalize)。如果我们设置为GL_TRUE，所有数据都会被映射到0（对于有符号型signed数据是-1）到1之间。
// 我们把它设置为GL_FALSE
// 步长(Stride)，它告诉我们在连续的顶点属性组之间的间隔 由于下个组位置数据在2个float之后 我们把步长设置为3 * sizeof(float)4
//  顶点数组缓存数据
// TODO 可以优化的地方
GLES30.glVertexAttribPointer(0, 2, GLES30.GL_FLOAT, false, 2 * 4, verBuffer)
// 现在我们已经定义了OpenGL该如何解释顶点数据，我们现在应该使用glEnableVertexAttribArray，以顶点属性位置值作为参数，启用顶点属性
GLES30.glEnableVertexAttribArray(0)
```

### 激活的着色器，绘制图元

好了现在开始我们最激动人心的一刻了

```
// 激活程序对象
// 在glUseProgram函数调用之后，每个着色器调用和渲染调用都会使用这个程序对象（也就是之前写的着色器)了
GLES30.glUseProgram(it)
// 第一个参数是我们打算绘制的OpenGL图元的类型
// 指定了顶点数组的起始索引
// 指定我们打算绘制多少个顶点
GLES30.glDrawArrays(GLES30.GL_TRIANGLES, 0, 3)
```

`glDrawArrays`是**顶点法**绘制。

	glDrawArrays方法进行物体的绘制,按照传入渲染管线顶点本身的
	顺序及选用的绘制方式将顶点组织成图元进行绘制的,也称之为顶点法.

顶点法弊端

	有些情况下不得不在顶点序列中出现很多重复的顶点,若希望减少重复顶
	点占用的空间,可考虑采用glDrawElements方法来进行物体的绘制.
	
这个问题在后面我们绘制纹理的时候会在代码层面有一个最直观的表现。

我们还可以使用`glDrawElements`**索引法**来绘制。

	要将顶点序列传入渲染管线,还需要将索引序列传入管线.
	绘制时管线根据索引值序列中的索引一一从顶点序列中去取出对应的顶
	点,再根据当前选用的绘制方式组织成图元进行绘制

所以使用`glDrawElements`**索引法**我们需要来定义个**索引的数组**，告诉OpenGL如何去取点。


**索引法绘制我们会在下节绘制纹理的时候在讲解**

完整代码地址

[LearnOpenGLES](https://github.com/flyminiboy/LearnOpenGLES)