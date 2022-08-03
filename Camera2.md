## Camera2

这篇文章主要的内容是介绍一些相机的基础理论知识和图和使用**camera2**，后续我们会专门写一篇文章从架构层面和源码的角度来分析**camera2**。

### 关于相机一些理论知识

首先我们先来看看摄像头的结构，从物理硬件层面来对摄像头有一个认识。

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wdAYXIE2NUgYF5V4t5zCd2c1Yl4HeNhNmGia1JCwWOCsm1n2LQcBSPFHEyBaQxicMicwUAju2ibkpx8jQ/0?wx_fmt=jpeg)

这幅图里面展示了摄像头模组的主要组成部分

|模组|解释|
|---|---|
| Lens Set | 镜头组，汇聚光线在CMOS/CCD上形成景物的图像 |
| Image sensor | 图像传感器（Camera sensor），负责将通过Lens的光信号转换为电信号，再经过内部AD转换为数字信号。两种主流传感器为CCD和CMOS |
| pixel | 像素，相机传感器上的最小感光单位。传感器上有许多感光单元，它们可以将光线转换成电荷，从而形成对应于景物的电子图像。 |
| DSP |Digital signal processor，数字信号处理器 |
| ISP |Image signal processor，图像信号处理器 |

下面我们还是通过一张表格来简单的认识一下这些模组是如何配合工作的

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdAYXIE2NUgYF5V4t5zCd2ceKQe2VjXGKrrDUQkhPPHKQ9n1hsu5BtgncBBFRyDMIonVibXeP1ZbLg/0?wx_fmt=png)

当然里面还有很多细节。本身我也不专业，所以这里就不展开赘述了。其实就是不会哈哈哈哈哈。

好了下面我们开始说一些和我们Android开发相关的内容。

#### 相机图像传感器（Camera Sensor）

这个就是上面的 **Image sensor** 。**Google Android developer** 称 **Camera Sensor**。这里我就和Google保持一致了。

真正和我们实际开发挂钩的主要是 **Camera Sensor 的旋转角度**。因为**相机图像传感器以传感器的自然方向（横向）输出其数据（图像缓冲区）**。所以这里就涉及到我们在**预览时候的画面角度**和**拍照图片旋转角度**。

#### 自然方向

**相机传感器**的自然方向是**横向**

**手机**的自然方向是**纵向**
	
#### Camera 预览方向

由于手机屏幕可以 360 度旋转，为了保证用户无论怎么旋转手机都能看到**正确的预览画面**。Android 系统底层根据**当前手机屏幕的方向对图像 Sensor 采集到的数据进行了旋转**处理后才传输给显示系统

使用**Camera1** 的时候，我们就需要使用 **Android** 系统提供一个 API 来手动设置 Camera 的**预览方向**，叫 `setDisplayOrientation`

**Camera2** 在预览时会**自动矫正预览角度**

#### 屏幕旋转角度（设备旋转）

表示设备从其**自然屏幕方向逆时针旋转**的角度值

> 例如，横向手机的设备旋转角度为 **90 度** 或 **270 度**（逆时针）

代码 `Display.getRotation() ` 返回的值，表示设备从其**自然屏幕方向逆时针旋转**的角度值。

#### 目标旋转角度

表示**顺时针旋转设备**使其达到**自然屏幕方向**需要旋转的度数

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wckgaTTzZfOwlGiaVenUI1SexJzOVekchfsjNTdWWadUI0iaxzTx3SDpmRua7ytFLaMptqxrjYxmnYg/0?wx_fmt=png)

**目标旋转角度** 和 **屏幕旋转角度** 是不同参考物不同维度的结论，刚好是一个互逆的过程。

#### 图片旋转角度

**图片旋转角度**描述的都是应如何**顺时针旋转**数据才能使其纵向显示

在 Android 中，**传感器方向被定义为一个常量值**，表示当设备处于自然位置时，相对于设备顶部，传感器旋转的角度（0、90、180、270）

**camera2 API** 包含一个 `SENSOR_ORIENTATION` 常量，获取指定摄像头的**传感器方向**

	前置摄像头的传感器方向为 270 度
	后置摄像头的传感器方向为 90 度

#### 方向计算

##### 传感器角度为90度

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wckgaTTzZfOwlGiaVenUI1SeMSCceBI89pxrqrr4bAowYgdw8Hmsz0zccnHLu5DVicsv1NIpTQnjB4w/0?wx_fmt=jpeg)

##### 传感器角度为270度

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wckgaTTzZfOwlGiaVenUI1SedB6X0cUWQFMSavSKzZNgIl139n6cEcrwDmnFj9kVjjfYpnmib5ic4eqg/0?wx_fmt=jpeg)

**传感器图像缓冲区**的整体旋转可以使用以下公式计算

```kotlin
rotation = (sensorOrientationDegrees - deviceOrientationDegrees * sign + 360) % 360
```

`sign`是**1**前置摄像头，**-1**后置摄像头

`sensorOrientationDegrees` 相机图像传感器角度

`deviceOrientationDegrees` 设备旋转角度

这部分请大家参考官网说明。

[https://developer.android.com/training/camerax/orientation-rotation?hl=zh-cn](https://developer.android.com/training/camerax/orientation-rotation?hl=zh-cn)

[https://developer.android.com/training/camera2/camera-preview](https://developer.android.com/training/camera2/camera-preview)

好好理解，这部分就和之前 OpenGL ES 里面的坐标系是一样的原理啊。将设备和图像通过旋转保证它们的显示一致。

### Camera2

#### 知识点

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wckgaTTzZfOwlGiaVenUI1Se2FvB2lQibjf3s1oe3plCNGJcP285DqvFtzzq5NvodXqMh85J8em0X3Q/0?wx_fmt=png)

#### 工作流程 

一张来自官网的大图

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdAYXIE2NUgYF5V4t5zCd2cx1Urdp8Ms6y2lfEO3T6JhyZuxibnekeMrq3K3d9xc9KgITCCTujicLnw/0?wx_fmt=png)

来自官网的一张图。略显复杂。小弟简单精简一下，请看下图。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wckgaTTzZfOwlGiaVenUI1Se4tsb1YOicIFSricjIlmtKOjILGia9FK7638fsNuhZ0nADJCXGdLm4t50A/0?wx_fmt=png)

**Camera2** 在 **Android设备**和**相机设备**之间设计了一个 **Pipeline**。`CameraCaptureSession` 就是这个 **Pipeline** 中的信使。

**Android设备**向**摄像头**发送 `CaptureRequest`，**摄像头**会返回
`CameraMetadata`

**单个 Android** 设备可以**有多个摄像头**。每个摄像头是一个 `CameraDevice`，一个`CameraDevice`可以同时输出多个流

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wdAYXIE2NUgYF5V4t5zCd2c0cR8OAbUgG60FGaqy8WvfBQ4314Z1aTxBdUp3blIoaia450ecqrSPzw/0?wx_fmt=png)

#### API 调用流程图

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wckgaTTzZfOwlGiaVenUI1SeF5XI3SyKzoNXzOFHibB1zxXtZHsbicPtiaqwOicM1vZQ5oQic9wKaaTO3Ag/0?wx_fmt=png)

#### 准备工作

在清单文件配置

```
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera.any" />
```

> 添加 **android.hardware.camera.any** 确保设备有摄像头
> 
> 指定 **.any** 意味着它可以是前置摄像头或后置摄像头


#### 开启相机

注意一定要**申请权限**，这些基本代码文章就不展示了，浪费篇幅。

```kotlin
private val cm by lazy {
    applicationContext.getSystemService(Context.CAMERA_SERVICE) as CameraManager
}

cm.openCamera("0", object : CameraDevice.StateCallback() {
    override fun onOpened(camera: CameraDevice) { // 摄像头已经打开
        
    }
    override fun onDisconnected(camera: CameraDevice) {
        
    }
    override fun onError(camera: CameraDevice, error: Int) {
        
    }
}, null)

```

#### 创建一个 CameraCaptureSession

这里我们配置了一个带有**两个输出缓冲区的相机会话**，一个属于  `TextureView` 做预览，另一个 属于 `ImageReader` 做拍照

```kotlin
val st = binding.mainTv.surfaceTexture
st?.let {
    surface = Surface(st)
    val targets = mutableListOf(surface)
    imageReader?.let {
        targets.add(it.surface)
    }
    cDevice?.createCaptureSession(targets, object : CameraCaptureSession.StateCallback() {
        override fun onConfigured(session: CameraCaptureSession) { // 会话通道配置成功
            
        }
        override fun onConfigureFailed(session: CameraCaptureSession) {
        }
    }, null)
}
```

#### 预览

```kotlin
val crb = camera.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW)
crb.addTarget(surface)
session.setRepeatingRequest(
    crb.build(),null,null
)
```

#### 拍照

```kotlin
val tprBuilder =
    camera.createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE) // 拍照模式
imageReader?.let { reader ->
    tprBuilder.addTarget(reader.surface)
}
// 拍照之前停止预览
session.stopRepeating();
session.capture(
    tprBuilder.build(),
    object : CameraCaptureSession.CaptureCallback() {
        override fun onCaptureCompleted(
            session: CameraCaptureSession,
            request: CaptureRequest,
            result: TotalCaptureResult
        ) {
            super.onCaptureCompleted(session, request, result)
            logE("take photo onCaptureCompleted")
            savePic()
            // 恢复预览
            _startPreview()
        }
    },
    null
)
```

#### 保存相片到本地

```kotlin
private fun savePic() {
    imageReader?.let { reader->
        val image = reader.acquireNextImage()
        val planes = image.planes
        val buffer = planes[0].buffer
        val data = ByteArray(buffer.remaining())
        buffer.get(data)
        image.close()
        val bitmap = BitmapFactory.decodeByteArray(data, 0, data.size)
        with(binding.mainPic) {
            visibility = View.VISIBLE
            // TODO 处理图片旋转问题
            setImageBitmap(bitmap)
        }
          bitmap.compress(Bitmap.CompressFormat.JPEG,100,fos)
    }
}
```

好了关于 Camera2 的使用我们就介绍这么多。关于 **Android camera**预计还会写俩篇，一篇会介绍 **Camera2 的架构**，一篇我们会介绍一下 `CameraX`

完整代码请点击这里啦

[https://github.com/flyminiboy/camera2](https://github.com/flyminiboy/camera2)




