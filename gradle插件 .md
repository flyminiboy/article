
## gradle-插件

一图镇楼

### 插件类型

* 脚本插件

* 对象插件

### 自定义插件的三种方式

1. 构建脚本
2. buildSrc项目
3. 独立项目

#### 构建脚本

直接在构建脚本中编写

将下面代码放在我们的 `build.gradle` 的构建脚本中测试

```
apply plugin:GreetingPlugin

class GreetingPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'hello gradle plugin'
            }
        }
    }

}
```

上面代码我们自定义了一个脚本，新建了一个 **hello** 的任务，任务内部打印了一句话 **hello gradle plugin**。

测试，执行任务
> ./gradlew hello

控制台打印。

```
> Task :app:hello
hello gradle plugin

BUILD SUCCESSFUL in 3s
1 actionable task: 1 executed

```

#### buildSrc项目

这个算是gradle内部约定的小玩意。

gradle文档有如下描述

> 当你运行Gradle时，它会检查项目中是否存在一个名为buildSrc的目录。然后Gradle会自动编译并测试这段代码，并将其放入构建脚本的类路径中。您不需要提供任何进一步的操作提示。
> 
> 对于多项目构建，只能有一个 buildSrc 目录，它必须位于项目根目录中

[https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#organizing_gradle_projects](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#organizing_gradle_projects)

1. 新建一个 **buildSrc** 目录。

2. 新建 **build.gradle** 或者 **build.gradle.kts** 文件

本示例以 **build.gradle.kts** 为演示代码

```
```

#### 独立项目

新建一个 `Java or kotlin Library`。



#### 扩展对象

自定义插件工作方式。 `ExtensionContainer`

#### 自定义任务

### 发布插件

参考[gradle-发布插件]()

```
gradlePlugin {
    plugins {
        modularPlugin {
            // Plugin id.
            id = 'xx'
            // Plugin implementation.
            implementationClass = 'x.x.x.xplugin'
        }
    }
}
```

这其实是 `Java Gradle Plugin` 提供的一个简化 API，其背后会自动帮我们创建一个 **[插件ID].properties** 配置文件，Gradle 就是通过这个文件类进行匹配的。如果你不使用 gradlePlugin API，直接手动创建 [插件ID].properties 文件，作用是完全一样的.

> main/resources/META-INF/gradle-plugins/xx.properties

* **xx.properties** 内容

```
implementation-class=x.x.x.xplugin
```

`Java Gradle Plugin` 源码地址。[https://github.com/gradle/gradle/blob/master/subprojects/plugin-development/src/main/java/org/gradle/plugin/devel/plugins/JavaGradlePluginPlugin.java](https://github.com/gradle/gradle/blob/master/subprojects/plugin-development/src/main/java/org/gradle/plugin/devel/plugins/JavaGradlePluginPlugin.java)


#### 应用插件


### 应用插件

#### 

#### 使用 plugin 分俩步

1. 解析（Resolve）： 找到包含给定插件的jar的正确版本，并将其添加到脚本类路径中
2. 应用（apply）：即调用 plugin

#### 写法分俩种

* `apply plugin`

> 解析和应用是分开的

* `plugins`

> 解析和应用是合并的


这种写法的前提是要使用的plugin是**核心plugin**或者发布在 [Gradle plugin repository](https://plugins.gradle.org/) 才可以！也就是说如果你自己写的gradle plugin或者公司的gradle plugin，除非是在插件仓库发布过的，否则只能使用第一种方式

原来我们在声明一个插件的时候一定需要在根 build.gralde下的 buildscripts 通过 dependencies 下，添加 classpath 来导入插件


pluginManagement

pluginManagement {} 只能在 settings.gradle 文件的第一块或者是 init.gradle 文件中



* `settings.gradle` 

```
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven { url('./repo') }
        google()
        mavenCentral()
    }
    // 导入插件库里面的插件
    plugins {

    }

    resolutionStrategy {

    }
}
dependencyResolutionManagement {
    // 仓库模式
//    FAIL_ON_PROJECT_REPOS
//      If this mode is set, any repository declared directly in a project, either directly or via a plugin, will trigger a build error.
//    PREFER_PROJECT
//      If this mode is set, any repository declared on a project will cause the project to use the repositories declared by the project, ignoring those declared in settings.
//    PREFER_SETTINGS
//      If this mode is set, any repository declared directly in a project, either directly or via a plugin, will be ignored.
//    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        maven { url('./repo') }
        google()
        mavenCentral()
    }
}
rootProject.name = "DocDemo"
include ':app'
include ':doc'
```

`plugins` 默认会从 [ Gradle Plugin Portal.](https://plugins.gradle.org/) 中解析插件。

`repositories` 自定义插件库 默认 `gradlePluginPortal()`

`resolutionStrategy` 自定义插件解析规则 ，例如更改请求的版本或明确指定实现构建坐标

```
    resolutionStrategy {
        eachPlugin {
            if (requested.id.namespace == 'com.example') {
                useModule('com.example:sample-plugins:1.0.0')
            }
        }
    }
```


* `init.gradle` 

```
settingsEvaluated { settings ->
    settings.pluginManagement {
        plugins {
        }
        resolutionStrategy {
        }
        repositories {
        }
    }
}
```


插件版本管理

插件解析规则

插件标记工件

`java-gradle-plugin`

应用二进制插件






下面我们做一个统计函数耗时的插件。





#### TransformClassesWithAsmTask

```kotlin
@get:Nested
abstract val visitorsList: ListProperty<AsmClassVisitorFactory<*>>
```





### 参考

* Gradle in Action
* [刚学会Transform，你告诉我就要被移除了](https://juejin.cn/post/7114863832954044446)
* [https://docs.gradle.org/nightly/userguide/plugins.html#sec:plugins_block](https://docs.gradle.org/nightly/userguide/plugins.html#sec:plugins_block)