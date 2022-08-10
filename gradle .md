## gradle 

### gradle 构建生命周期

* 初始化阶段

* 配置阶段

* 执行阶段

**初始化阶段**

### groovy kotlin

### 包装器

wapper

### 构建块

#### Project

#### Task

##### 动作（action）

> 就是在 task 中放置构建逻辑

* doFirst
* doLast （左移操作符 << 可以作为 doLast 的快捷方式）

##### 依赖

`dependsOn` 允许声明依赖一个或多个任务

##### 输入和输出

##### task 配置块

task 配置块 永远在 task 动作之前执行

#### property

##### 扩展属性

ext

##### gradle 属性

`gradle.properties` 文件中声明

### Task 任务

### Plugin 插件

#### 插件类型

##### 脚本插件

##### 对象插件

#### 如何应用插件

##### 使用 plugin 分俩步

1. 解析（Resolve）： 找到包含给定插件的jar的正确版本，并将其添加到脚本类路径中
2. 应用（apply）：即调用 plugin

##### 写法分俩种

* `apply plugin`

> 解析和应用是分开的

* `plugins`

> 解析和应用是合并的


这种写法的前提是要使用的plugin是核心plugin或者发布在[Gradle plugin repository才可以！也就是说如果你自己写的gradle plugin或者公司的gradle plugin，除非是在插件仓库发布过的，否则只能使用第一种方式


#### 自定义插件的三种方式

1. 构建脚本
2. buildSrc项目
3. 独立项目

```
gradlePlugin {
    plugins {
        modularPlugin {
            // Plugin id.
            id = 'xx.xx.xx'
            // Plugin implementation.
            implementationClass = 'x.x.x.xplugin'
        }
    }
}
```

这其实是 `Java Gradle Plugin` 提供的一个简化 API，其背后会自动帮我们创建一个 **[插件ID].properties** 配置文件，Gradle 就是通过这个文件类进行匹配的。如果你不使用 gradlePlugin API，直接手动创建 [插件ID].properties 文件，作用是完全一样的.

> main/resources/META-INF/gradle-plugins/neudoc.properties

这个类似 Google 的 `AutoService`

#### 发布插件

这个地方需要注意，gradle 7.x 移除了 maven 插件，推荐使用 maven-publish 插件。


### 依赖




### 参考

* Gradle in Action
* [刚学会Transform，你告诉我就要被移除了](https://juejin.cn/post/7114863832954044446)