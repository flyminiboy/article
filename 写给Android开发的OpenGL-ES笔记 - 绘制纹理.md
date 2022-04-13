## 写给Android开发的OpenGL-ES笔记 - 绘制纹理


这篇文章我们来绘制纹理。既然要绘制纹理，首先我们肯定要知道什么是**纹理**。不然最后绘制个寂寞啊

**这里我们只讨论2D纹理**

### 纹理

我们先通过一个比喻来说说明一下**纹理**。富有的小伙伴们买了新房子，希望给电视墙上来点酷酷的图案。现在我们有如下俩个选择：

1. 自己或找设计师直接在墙面做绘制
2. 我们贴一个壁纸

方法1直接绘制，很明显有难度呢。方法2贴壁纸就简单多了，其实**纹理**就可以想象成这里的壁纸。

**2D纹理是一个图像的二维数组，用2D纹理渲染时，纹理坐标用作纹理图像中的索引**

**纹理坐标在x和y轴上，范围为0到1之间**（注意我们使用的是2D纹理图像），一般用**（s,t）**指定，有时也称作**（u,v）**坐标。


使用**纹理坐标获取纹理颜色**叫做**采样**（Sampling）

**纹理坐标起始于(0, 0)，也就是纹理图片的左下角，终始于(1, 1)，即纹理图片的右上角**，这个坐标很重要。后续好多的知识都是要依赖这个坐标的。关于坐标我们后续会专门在一篇文章进行说明。这里大家就记住这个就行啊，有些知识是需要背的，没有道理的。

**纹理坐标这个结论是各种权威书籍和网站都是这么确定的，大家不要被网上一些不明所以然的文章给搞晕了。**

**参考**

**LearnOpenGL 网站** 和 **OpenGL ES 3.0 编程指南**

**纹理坐标起始于(0, 0)，也就是纹理图片的左下角，终始于(1, 1)，即纹理图片的右上角**

**纹理坐标起始于(0, 0)，也就是纹理图片的左下角，终始于(1, 1)，即纹理图片的右上角**

**纹理坐标起始于(0, 0)，也就是纹理图片的左下角，终始于(1, 1)，即纹理图片的右上角**

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdEiciaE6iaj6cr4QoOwVbj12qJcEQFI7XK0MQjMBeCbib5qKiathSgnXPNPoGqeur1WpabuswPYpZxUfA/0?wx_fmt=png)

**一个纹理的单独数据元素称作纹素（Texture Pixel，也叫Texel）**

	Texture Pixel也叫Texel，你可以想象你打开一张.jpg格式图片，不断
	放大你会发现它是由无数像素点组成的，这个点就是纹理像素；
	注意不要和纹理坐标搞混，纹理坐标是你给模型顶点设置的那个数组，
	OpenGL以这个顶点的纹理坐标数据去查找纹理图像上的像素，
	然后进行采样提取纹理像素的颜色。

理论知识就介绍这些了。我每次习惯介绍这些理论知识是因为我发现自己一路摸索学习之后，觉得只有真正的懂的理论知识，再去实践才真的有悟的感觉，对知识的掌握有了质的升华。其次也是方便大家再看了我的不懂以后，再去看其他的文章，对一些知识有一些基础的储备，不至于一脸懵逼。


### 将一张本地图片绘制到屏幕

小二上代码，来了大爷。

#### 顶点着色器

```
#version 300 es

layout (location = 0) in vec4 vPosition;
layout (location = 1) in vec2 tPosition; // 变化1️⃣，这是定义了输入的纹理坐标

out vec2 outTexturePosition; // 变化2️⃣，这里定义了输出的纹理坐标

void main() {
    gl_Position = vPosition; // gl_Position 内建变量，是一个vec4类型
    outTexturePosition = tPosition;
}
```

#### 片段着色器

```
#version 300 es
precision mediump float;

in vec2 outTexturePosition; // 变化1️⃣，这里定义了输入纹理坐标来自顶点着色器的输出

uniform sampler2D sTexture; // 变化2️⃣，纹理采样器 （获取对应的纹理ID）

out vec4 outColor;

void main() {
    outColor = texture(sTexture, outTexturePosition); //变化3️⃣ texture函数会使用之前设置的纹理参数对相应的颜色值进行采样。这个片段着色器的输出就是纹理的（插值）纹理坐标上的(过滤后的)颜色
}
```

`sampler2D`是我们接触到的第一个**纹理采样器**

后续我们在渲染相机数据的时候会使用到另一个**纹理采样器**`samplerExternalOES`

#### 顶点坐标

```
val vertexCoords = floatArrayOf(
    -1.0f, -1.0f, // 左下
    1.0f, -1.0f, // 右下
    1.0f, 1.0f, // 右上
    -1.0f, 1.0f // 左上
)

val vertexCoordsBuffer by lazy {
    ByteBuffer.allocateDirect(vertexCoords.size * 4)
        .order(ByteOrder.nativeOrder())
        .asFloatBuffer().apply {
            put(vertexCoords)
            position(0)
        }
}
```

#### 纹理坐标

```
val textureCoords = floatArrayOf(
    0.0f, 0.0f, // 左下
    1.0f, 0.0f, // 右下
    1.0f, 1.0f, // 右上
    0.0f, 1.0f // 左上
)

val textureBuffer by lazy {
    ByteBuffer.allocateDirect(textureCoords.size * 4)
        .order(ByteOrder.nativeOrder())
        .asFloatBuffer().apply {
            put(textureCoords)
            position(0)
        }
}
```

#### 生成纹理ID引用

```
val textures = intArrayOf(1)

// 第一个参数 生成纹理的数量
// 第二个参数 储存纹理ID
GLES30.glGenTextures(1, textures, 0)
```

#### 绑定纹理ID

```
GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, textures[0])
```

#### 为当前绑定的纹理对象设置环绕、过滤方式

```
GLES30.glTexParameteri(GLES30.GL_TEXTURE_2D, GLES30.GL_TEXTURE_WRAP_S, GLES30.GL_REPEAT)
GLES30.glTexParameteri(GLES30.GL_TEXTURE_2D, GLES30.GL_TEXTURE_WRAP_T, GLES30.GL_REPEAT)
GLES30.glTexParameteri(
    GLES30.GL_TEXTURE_2D,
    GLES30.GL_TEXTURE_MIN_FILTER,
    GLES30.GL_LINEAR
)
GLES30.glTexParameteri(
    GLES30.GL_TEXTURE_2D,
    GLES30.GL_TEXTURE_MAG_FILTER,
    GLES30.GL_LINEAR
)
```

#### 纹理环绕方式

这个就和我们在设置桌面的时候的背景模式，平铺，重复。。。一个意思。


|环绕方式|	描述|
|  ----  | ----  |
|`GL_REPEAT`|对纹理的默认行为。重复纹理图像。|
|`GL_MIRRORED_REPEAT`|	和`GL_REPEAT`一样，但每次重复图片是镜像放置的。|
|`GL_CLAMP_TO_EDGE`|	纹理坐标会被约束在0到1之间，超出的部分会重复纹理坐标的边缘，产生一种边缘被拉伸的效果。|
|`GL_CLAMP_TO_BORDER`|	超出的坐标为用户指定的边缘颜色。|

盗图一张

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdEiciaE6iaj6cr4QoOwVbj12qib4DTB6JhRAm4ZP6rO1ibsTIxRniaIJVHs6w4X8dohu19wzo5Tt3fN87A/0?wx_fmt=png)

#### 纹理过滤

**纹理坐标**不依赖于**分辨率(**Resolution)，它可以是任意浮点值，所以OpenGL需要知道怎样将**纹理像素映射到纹理坐标**

**纹理过滤**有很多个选项，但是现在我们只讨论最重要的两种：`GL_NEAREST`和`GL_LINEAR`

`GL_NEAREST`（也叫**邻近过滤**，Nearest Neighbor Filtering）是OpenGL默认的纹理过滤方式。当设置为GL_NEAREST的时候，OpenGL会选择中心点最接近纹理坐标的那个像素。

`GL_LINEAR`（也叫**线性过滤**，(Bi)linear Filtering）它会基于纹理坐标附近的纹理像素，计算出一个插值，近似出这些纹理像素之间的颜色。

当进行**放大(Magnify)**和**缩小(Minify)**操作的时候可以设置**纹理过滤**的选项

盗图一张说明问题。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdEiciaE6iaj6cr4QoOwVbj12qzFZoHCSBm6kyO51ibibIyXK2ib52u9xSWto3oRdqj1OT3sS5TecPc9wMA/0?wx_fmt=png)

	GL_NEAREST产生了颗粒状的图案，我们能够清晰看到组成纹理的像
	素，而GL_LINEAR能够产生更平滑的图案，很难看出单个的纹理像素
	
**这部分内容摘自 https://learnopengl-cn.github.io/**

#### 将bitmaph转换纹理

```
val bitmap = BitmapFactory.decodeResource(surfaceView.context.resources, R.drawable.a)

GLUtils.texImage2D(GLES30.GL_TEXTURE_2D, 0, bitmap, 0)
GLES30.glGenerateMipmap(GLES30.GL_TEXTURE_2D)
```

#### 激活纹理

```
glActiveTexture(GL_TEXTURE0); // 在绑定纹理之前先激活纹理单元 纹理单元GL_TEXTURE0默认总是被激活
glBindTexture(GL_TEXTURE_2D, texture);
```

#### 绘制

```
GLES30.glDrawArrays(GLES30.GL_TRIANGLE_FAN, 0, 4
```

这个我们使用了**顶点法**绘制，大家对绘制不熟悉可以参考上一pain文章。[写给Android开发的OpenGL-ES笔记 - 绘制矩形](https://mp.weixin.qq.com/s?__biz=Mzg3NzU3OTQwOA==&amp;mid=2247483813&amp;idx=1&amp;sn=3f838f009f37d315ab05d4c2eafbd670&amp;chksm=cf219c5af856154cb0e9bf4153b01d1c405966721055df51b40c0f3ba2848ecae9e2952b72b0&token=2062737436&lang=zh_CN#rd)

好了关于绘制纹理就介绍到这里。

**如果大家最终实现了效果，可以发现一个问题，图片倒了。这个问题我们在下一篇文章OpenGL坐标世界来详细说明问题来源和如何解决问题。**

完整代码请参考
[https://github.com/flyminiboy/LearnOpenGLES](https://github.com/flyminiboy/LearnOpenGLES)


