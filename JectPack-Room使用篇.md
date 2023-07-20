## Jenkins Android CI/CD

### 安装Jenkins

```
brew install jenkins
```

#### 启动Jenkins

```
brew services start jenkins
```

#### 停止Jenkins 

```
brew services stop jenkins
```

#### 重启Jenkins

```
brew services restart jenkins
```

#### 插件

1. 推荐插件

2. 自定义插件

### 环境搭建

#### JDK

#### Android SDK

#### Gradle

1. 自动安装
2. 指定本地目录

```
/Users/fly/.gradle/wrapper/dists/gradle-7.4-bin/c0gwcg53nkjbqw7r0h0umtfvt/gradle-7.4
```

对接飞书

### 创建任务

#### 自由风格

#### 流水线

1.  执行shell
```
curl -X POST -H "Content-Type: application/json" \
	-d '{"msg_type":"text","content":{"text":"Jenkins打包成功"}}' \
  https://open.feishu.cn/open-apis/bot/v2/hook/56f25b17-0860-4ca5-bc5a-bc4bc0534e17
```

2. 飞书插件

https://www.larksuite.com/hc/zh-CN/articles/360038580574

 fuck下架了

https://www.cnblogs.com/my-blogs-for-everone/p/11364629.html