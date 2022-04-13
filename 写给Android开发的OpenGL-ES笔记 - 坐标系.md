## 写给Android开发的OpenGL-ES笔记 - 坐标系

这篇文章我们重点来介绍**坐标系**，然后我们把之前绘制倒的纹理图像，给他来个斗转星移大法让他显示正常，变形的不好看的样子我们也给他修正了。

我们今天介绍的所有坐标系都是**笛卡尔坐标系（也称直角坐标系）**

**二维的直角坐标系**是由两条相互垂直、相交于原点的数线构成的。通常由两个互相垂直的坐标轴设定，通常分别称为**x-轴**和 **y-轴**，两个坐标轴的相交点，称为**原点**，通常标记为**O**。

而我们今天要说的这些坐标系的区别在于**原点的选择**和**y轴正方向的选定**。

#### 规范化设备坐标（OpenGL 坐标）

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfdc7P6uvOIQSupCeLcUMXbssXsdcqp7sia8OkSyzibtEnMrvmaiapm79DCTohlmWuiaN8u27wKjpnwQA/0?wx_fmt=png)

这个就是一个规范，名字就体现了。本身不会影响图像的展示。

#### 纹理坐标

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdEiciaE6iaj6cr4QoOwVbj12qJcEQFI7XK0MQjMBeCbib5qKiathSgnXPNPoGqeur1WpabuswPYpZxUfA/0?wx_fmt=png)

#### 屏幕坐标

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfdc7P6uvOIQSupCeLcUMXbPXyvNNlicSZtEW2QHU95XmicGIh6gdiaHewKwNJQbsVNfzeEz1tm5y2zw/0?wx_fmt=png)

通过图我可以得到四个小结论

1. 可以看到上面是三个坐标，它们的原点选择都不同。**规范化设备坐标**和**纹理坐标**的坐标值**-1.0到1.0**。

2. **规范化设备坐标**和**屏幕坐标**它们的目标是一致的都是我们的**设备屏幕**。

3. **纹理坐标**的目标是**纹理**。

4. **纹理坐标**和**屏幕坐标**的目标不一致，最重要的是它们的**y轴的方向是相反**的。

这四个结论很重要，大家一定要自己琢磨明白。

现在我们还是通过之前的例子来把这个抽象的理念具化一下下。

我们要给电视墙上面装饰壁纸。比如电视墙宽5米，高3米。

**规范化设备坐标**和**屏幕坐标**只是俩个不同的维度来描述这个电视墙。这时一些心思缜密的小伙伴可能已经发现了一个新问题。o my god , fuck.老问题还没有明白，又来了一个新问题。

**我们规范化设备坐标是一个正方向。我们的墙是一个矩形啊。？？？？**

这个问题我们后面会有专门有一篇文章进行说明😄。现在我们还是回到当前的问题。

你可以用屏幕坐标的维度说(0,0)是墙的左上角，也可以通过规范化设备坐标的维度说(-1,-1)是墙的左上角，他俩是不冲突的，只是不同的维度去描述同一个位置而已。

**纹理**，也就是现在我们的壁纸却不一样了，我们说纹理的(0,0)是壁纸上面的左下角的坐标。如下图

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfdc7P6uvOIQSupCeLcUMXbbzHP7q70obPiaJiaGaSXqDLUYzOe5HZwhfhZ2xKTzvzicpQwypsL7U0Fw/0?wx_fmt=png)

现在我们要把这张图片贴到电视墙上了。如下图

### 上代码

现在我们来互看之前绘制纹理的时候定义的顶点坐标。

```
val vertexCoords = floatArrayOf(
    -1.0f, -1.0f, // 左下
    1.0f, -1.0f, // 右下
    1.0f, 1.0f, // 右上
    -1.0f, 1.0f // 左上
)
```

再看看之前我们是按照顶点坐标的定义顺序去定义的纹理坐标。

```
val textureCoords = floatArrayOf(
    0.0f, 0.0f, // 左下
    1.0f, 0.0f, // 右下
    1.0f, 1.0f, // 右上
    0.0f, 1.0f // 左上
)
```

这俩个组合得到的映射

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfdc7P6uvOIQSupCeLcUMXbAPhwicUvb2LspvvCyJdpckj53tLluo0icNv5PbspeiarZHTwc1iavIKWrQ/0?wx_fmt=png)

哎呀呀，好顺眼啊。请理智。注意我们标注的**y轴正方向**，什么是正方向。想象一下，按照这个正方向，我们的图片是不是就倒了。想象不到你就把这个图旋转180度，让正方向符合你视觉效果，再看图片是不是倒了。

这就是之前我们绘制的结果。

现在我们看看正确的应该是什么样子的

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfdc7P6uvOIQSupCeLcUMXb9hic4ktoiaZm9amiaibrTATQGZicLH5trBVHPvibGnMiaGS2PaIAsbiaV1cINA/0?wx_fmt=png)

来重新定义纹理坐标

```
val textureCoords = floatArrayOf(
    1.0f, 1.0f, // 右上
    0.0f, 1.0f, // 左上
    0.0f, 0.0f, // 左下
    1.0f, 0.0f // 右下
)
```

好了完美运行。

**这部分知识后续我们在渲染相机数据也是要用到的，相机的前置和后置摄像头的旋转问题以及前置摄像头镜像问题。**

起始镜像也很好理解我们现场演示一下。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfdc7P6uvOIQSupCeLcUMXb3g0Jrl1t3aoPpcf9xAgnLJ7dmoLmtAicjY7E4AkDiaqGKdf9NY8Cwn5A/0?wx_fmt=png)

根据图片我们来定义**纹理坐标**

```
val textureCoords = floatArrayOf(
    0.0f, 1.0f, // 左上
    1.0f, 1.0f, // 右上
    1.0f, 0.0f, // 右下
    0.0f, 0.0f // 左下
)
```

现在我们就可以展示出来一个镜像的图片了。

**我这里所有的纹理坐标定义都是基于我前面顶点坐标在规范化设备坐标的一个顺序啊，这个地方大家在自己实践的时候要注意一下。**

### 结语

小弟这篇文章可能和网上大多数的文章表现不一致，这个我觉得看自己的理解了。我记录的是我理解的在我需要的的一个场景下的一个学习笔记。如果有误导大家的地方，请不要喷。

如果我后续有更多的时间，也许会写一个关于这部分的纯理论补充内容。

	局部坐标(Local Coordinate)
	世界坐标(World Coordinate)
	观察坐标(View Coordinate)
	裁剪坐标(Clip Coordinate)
	屏幕坐标(Screen Coordinate)

最后的最后附上完整的项目地址

[https://github.com/flyminiboy/LearnOpenGLES](https://github.com/flyminiboy/LearnOpenGLES)






