## Android签名系列-基础

### 生成签名文件

#### Android studio

**直接看图吧，保姆级**

**直接看图吧，保姆级**

**直接看图吧，保姆级**

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZo7t3aMRRTsmP4FayrvJj1LGnG4TcDdkQsq1PD8WT4OKvjtpNQbc0xg/0?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZWicUMibP2Ur9WOtkzJGKP59Gas6IXRTI5FOCqkcS6K0iasZK7nIwJ7zwA/0?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZt69nsfjThxVgPpso3kFIy0ZjjKa2tD2oh2LQvAvGRcqDjqszzTRMmQ/0?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZNu7UVhJIsIe0mmaB9hXjbSdJKweBUjMqiaBEE899py5NKaTR9kNGkRA/0?wx_fmt=jpeg)

#### 命令行-`keytool`

Keytool 是一个Java 数据证书的管理工具。

下面举例一些Android开发平时用到比较用的选项 

	-keyalg 指定密钥算法(默认DSA)，一般写RSA
	-sigalg 指定签名算法
		keyalg = RSA 时，签名算法有：MD5withRSA、SHA1withRSA、SHA256withRSA（默认）、SHA384withRSA、SHA512withRSA
		keyalg = DSA 时，签名算法有：SHA1withDSA、SHA256withDSA（默认）
	-genkey 创建一个“xxx.keystore”文件
	-alias 别名(默认mykey)，这个alias通常不区分大小写
	-keystore 指定密钥库路径名称
	-dname 指定证书拥有者信息
		CN=名字与姓氏,OU=组织单位名称,O=组织名称,L=城市或区域名称,ST=州或省份名称,C=单位的两字母国家代码
	-storepass 密钥库密码
	-keypass 设置key的密码
	-validity 有效期(默认90)，单位天
	-storetype JKS|PKCS12(指定密钥库的类型，可用类型为：JKS、PKCS12等。jdk9以前，默认为JKS。自jdk9开始，默认为PKCS12)
	-list 显示密钥库中的证书信息
	-v 详细输出，显示密钥库中的证书详细信息


通过下面命令生成签名文件
	
```
keytool -genkey -keyalg RSA  -keystore test.keystore -validity 3650 -storepass 123456 -keypass 123456
```

**上图**

**上图**

**上图**

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZicibAxEPTdYXj01dgrVqKATyibGx91ylotSdkdcp2ErEkibcRFx19ibNhSQ/0?wx_fmt=jpeg)

#### 版本

1.  v1 基于 JAR 签名
2. v2 Android 7.0引入
3. v3 Android 9.0引入

更加详细的介绍可能会在下一篇😂😂😂😂😂😂

### 签名

#### Android studio

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZk8T6GiaXg1JuyFm7gXtRzH5icS2oc9rcHLTrj0rf2HpGdMWibK6mod6Hw/0?wx_fmt=jpeg)

根据自己的需求选择生成**AAB**还是**apk**

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZWicUMibP2Ur9WOtkzJGKP59Gas6IXRTI5FOCqkcS6K0iasZK7nIwJ7zwA/0?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZFW8mnsiar8rUr4K0Wq3nQD05qULkxcK8jXdEEibncNH79uRZx9yF7sSA/0?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZqT9tfICpShXhTJ9Fhe7tMRpmJicib1icnQj1HamvaZkGV18V3qo6RtqbA/0?wx_fmt=jpeg)

#### 工具

* jarsigner
* apksigner

#### jarsigner

jarsigner是JDK提供的针对jar包签名的通用工具

#### apksigner

apksigner是Google官方提供的针对**Android apk**签名及验证的专用工具, 记住一定是针对**Android apk**的，比如我们发布Google play打包的**aab**是**无法使用apksigner**这个工具的。

**注意：您不能使用 apksigner 为 app bundle 签名。**

**注意：您不能使用 apksigner 为 app bundle 签名。**

**注意：您不能使用 apksigner 为 app bundle 签名。**

关于AAB有可能我们会在单独一篇文章中进行讲解的😂😂😂😂😂😂

#### 查看签名-直接查看签名文件

```
keytool -list -v -keystore xxx(签名文件)
```

需要输入密钥库密码，上面在生成的时候 `-storepass` 指定的。

**翠花上图**

**翠花上图**

**翠花上图**

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZkG7JpQYUZN2v2tr6gzBVLicB6mc2CCJiaDnLbbet5kPwJ7WSSAS9vnAA/0?wx_fmt=jpeg)

#### 查看签名-查看apk

```
apksigner verify -v --print-certs xxx(apk)
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZurt3ajkGVMY6uJKBD1QtLIiaqT5HqBHicyblaJib0CAV4NbCvKpl4ibm9A/0?wx_fmt=jpeg)

```
keytool -printcert -jarfile xxx(apk)
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/ibExRe3rl9wc1libRQB35O2mm8BqfzS6EZDUsVueM2jLOicXDlS3FibnWdyfAQdEd3PF1AHajWsxjsiclkQfnrTibuhw/0?wx_fmt=jpeg)


#### 签名

```
 jarsigner -keystore debug.keystore MyApp.apk androiddebugkey
```


jarsigner -verbose -keystore 签名文件路径 -digestalg SHA-256 -sigalg SHA256withRSA -storepass 密码 -keypass 密码 aab文件路径 别名(key alias)


apk

1. 对齐
2. 重新签名


####  zipalign 

zipalign -p -f -v 4 app/build/outputs/apk/debug/app-debug.apk align.apk

zipalign -c -v 4 outfile.apk

#### 签名

apksigner sign --ks app/keystore/asgen --ks-key-alias askey --ks-pass pass:123456 --v2-signing-enabled true -v --out new.apk app/build/outputs/apk/debug/app-debug.apk


### 签名校验

### 签名完整性校验



