## Android反编译-基础

### apktool

通过**Homebrew**安装

```
brew install apktool
```

#### 下载链接


[https://ibotpeaches.github.io/Apktool/install/](https://ibotpeaches.github.io/Apktool/install/)

### dex2jar

#### 下载链接

[https://sourceforge.net/projects/dex2jar/files/](https://sourceforge.net/projects/dex2jar/files/)

解压缩在 mac 中使用 **d2j-dex2jar.sh**

```
sh d2j-dex2jar.sh classes.dex
```

一路上斩妖除魔，第一关

	d2j-dex2jar.sh: line 36: ./d2j_invoke.sh: Permission denied

解决办法

```
chmod 777 d2j_invoke.sh
```

一路上斩妖除魔，第二关

	com.googlecode.d2j.DexException: not support version.
		at com.googlecode.d2j.reader.DexFileReader.<init>(DexFileReader.java:151)
		at com.googlecode.d2j.reader.DexFileReader.<init>(DexFileReader.java:211)
		at com.googlecode.dex2jar.tools.Dex2jarCmd.doCommandLine(Dex2jarCmd.java:104)
		at com.googlecode.dex2jar.tools.BaseCmd.doMain(BaseCmd.java:288)
		at com.googlecode.dex2jar.tools.Dex2jarCmd.main(Dex2jarCmd.java:32)

定位问题
	
#### DexFileReader

```
    public DexFileReader(ByteBuffer in) {
        in.position(0);
        in = in.asReadOnlyBuffer().order(ByteOrder.LITTLE_ENDIAN);
        int magic = in.getInt() & 16777215;
        if (magic == 7890276) {
            int version = in.getInt() & 16777215;
            if (version != 3486512 && version != 3552048) {
                throw new DexException("not support version."); // 报错位置
            } else {
                skip(in, 32);
                int endian_tag = in.getInt();
                if (endian_tag != 305419896) {
                    throw new DexException("not support endian_tag");
                } else {
                    // 省略非关键代码
                }
            }
        } else if (magic == 7955812) {
            throw new DexException("Not support odex");
        } else {
            throw new DexException("not support magic.");
        }
    }
```

打开要反编译的dex文件（二进制）

推荐Mac上一个小工具 **Hex Firend** [https://hexfiend.com/](https://hexfiend.com/)

也可以使用 **010 Editor**

```
6465 780a 3033 3700 066e 6ce2 0cea b6b0
```

#### 什么是dex文件

**.java->.class->.dex**

魔数（class文件结构中也有这个东西），Dex头部校验

解决办法

[https://blog.csdn.net/UFO_SERIESOFSOFT/article/details/106436557](https://blog.csdn.net/UFO_SERIESOFSOFT/article/details/106436557)

### jd-gui 


[https://github.com/tofi86/universalJavaApplicationStub/blob/v3.0.6/src/universalJavaApplicationStub](https://github.com/tofi86/universalJavaApplicationStub/blob/v3.0.6/src/universalJavaApplicationStub)


### 参考

[https://www.cnblogs.com/librarookie/p/16364384.html](https://www.cnblogs.com/librarookie/p/16364384.html)

[https://developer.android.com/studio/publish/app-signing?hl=zh-cn](https://developer.android.com/studio/publish/app-signing?hl=zh-cn)

[https://source.android.com/docs/security/features/apksigning?hl=zh-cn](https://source.android.com/docs/security/features/apksigning?hl=zh-cn)

[https://source.android.com/docs/core/dalvik/dex-format?hl=zh-cn#dex-file-magic](https://source.android.com/docs/core/dalvik/dex-format?hl=zh-cn#dex-file-magic)

[https://developer.android.com/studio/command-line/zipalign?hl=zh-cn](https://developer.android.com/studio/command-line/zipalign?hl=zh-cn)