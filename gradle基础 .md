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
