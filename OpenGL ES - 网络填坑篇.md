## OpenGL ES - 网络填坑篇

这一路学习 `OpenGL ES` 看了不少网上的文章，有的是真敢写啊。小弟真的怀疑它们代码是真的运行通过的代码贴出来的。主要问题就是**OpenGL ES版本**的一些变动，引发的血案。

下面我们就开始网络打假行动，当然也许它们真的是运行通过的代码，那么就怪我才疏学浅了，乱打了。

### 环境

下面是小弟代码运行和测试环境。

1.  OpenGL ES 3.0 
2. Android 版本6.0，8.1.0以及HarmonyOS 2.0.0
3. MacOS Monterey 12.3.1 (M1)
4. Android Studio 2021.1.1 Patch 3

### 问题

#### 版本声明

当我们要使用 OpenGL ES  的时候，一定要在我们的 GLSL 文件中**第一行**声明版本。好习惯。

例如当我们要使用3.0版本时

```
#version 300 es
```

#### 布局限定符

`layout`

我们可以通过` layout (location = 0) ` 设定输入变量的位置值（Location）。

这样我们就可以省略 `GLES30.glGetAttribLocation(mProgram, "vPosition")` 函数的使用。

`GLES30.glVertexAttribPointer(0, 2, GLES30.GL_FLOAT, false, 2 * 4, mVertexBuffer)`

第一个参数就是我们的输入变量的位置值（Location）。

#### 输入输出

**OpenGL ES 3.0**中将2.0的`attribute`改成了`in`，**顶点着色器**的`varying`改成`out`，**片段着色器**的`varying`改成了`in`，也就是说**顶点着色器的输出就是片段着色器的输入**

顶点着色器的输出和片段着色器的输入，不能使用**布局限定符**。

当**片段着色器有多个渲染目标**（MRT）时，可以使用**布局限定符**指定每个输出前往的渲染目标

#### 内建变量

**OpenGL ES 2.0**的`gl_FragColor`和`gl_FragData`在**OpenGL ES 3.0**中取消掉了，需要自己定义`out`**变量**作为**片段着色器的输出颜色**

#### 函数

**OpenGL ES 3.0**的**shader**中`texture2D`和`texture3D`取消掉了，全部使用`texture`替换

#### 扩展纹理
 
 扩展纹理 使用 `GL_OES_EGL_image_external_essl3`不是 `GL_OES_EGL_image_external`
 
源码地址

[https://github.com/flyminiboy/camera2](https://github.com/flyminiboy/camera2)

[https://github.com/flyminiboy/LearnOpenGLES](https://github.com/flyminiboy/LearnOpenGLES)

就水这么一篇文章吧。努力，我还能写。

 
 
 
 
 
 


