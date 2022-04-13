## Gerrit 

### Gerrit 是什么

**Gerrit**是**Google**为Android系统研发量身定制的一套免费开源的**代码审核系统**，它在传统的源码管理协作流程中强制性引入代码审核机制，通过人工代码审核和自动化代码验证过程，将不符合要求的代码屏蔽在代码库之外，确保核心代码多人校验、多人互备和自动化构建核验。


so **Gerrit ≠ Git**

通过俩张图来说明

#### 没有使用Gerrit代码管理结构

![](https://ywp3ec89p0.feishu.cn/file/boxcn2qzuKCD5RsSMIGCC7TrKEc)

#### 使用Gerrit代码管理结构

![](https://ywp3ec89p0.feishu.cn/file/boxcn0SsXF8JgiRcR4EgLiNp0Uc)

**开‎发人员的修改首先将上载到 Gerrit, 但 实际上并不成为项目的一部分, 直到它们被审阅和接受。**

### Gerrit人员角色配置

使用**OpenID**登录，第一个登录的用户为**admin**，创建**dev**帐号、**review**帐号和**verify**帐号，创建**dev**、**review**和**verify**用户组并添加相应用户

### 使用Gerrit

[注册openID](https://git.yeecall.com)

#### 设置Username（代码同步时需要用到）

![](https://ywp3ec89p0.feishu.cn/file/boxcnbyczb8VNGrnEZQrEm4UZSh)

![](https://ywp3ec89p0.feishu.cn/file/boxcnXrDF2gQmPGp8ZVkOGuc0jc)

#### 克隆工程

* HTTP模式

设置 **HTTP Password** 路径账户 - ->> Settings -->> HTTP Password 处获取

![](https://ywp3ec89p0.feishu.cn/file/boxcnc0HJwUT8Q6t7RbMhW7rNkd)

* SSH模式

设置 **SSH Public Keys** 路径账户 - ->> Settings -->> SSH Public Keys 处配置

![](https://ywp3ec89p0.feishu.cn/file/boxcn8HguBuCrPDApNmfzRIJXBe)

#### 克隆

**commit-msg**，提供自动写入 **Change-Id** 至 **git log** 内功能

![](https://ywp3ec89p0.feishu.cn/file/boxcnhk1LcPFOi3RnJTA5OyWQic)

否则在提交代码的时候出现如下错误

> missing Change-Id in commit message footer

因为 **Gerrit** 要求每个提交都要有 **Change-Id** 

**Change-Id** 是 **Gerrit** (代码审核平台)的概念, 与 **git **(版本管理) 是没有关系的，是通过 **Git hook **机制实现

#### 提交代码

`git push origin HEAD:refs/for/develop`

**refs/for** 意义在于我们提交代码到服务器之后是需要经过**code review** 之后才能进行真正代码入库

**develop** 表示远端分支

### 最后直接上一个Android官网给的一个 Gerrit 的工作流程图

![](https://ywp3ec89p0.feishu.cn/file/boxcnhmBK6HZMZXOt9kM5LAY1RL)


修改 .gitmodules 文件中对应模块的url属性;
使用 git submodule sync 命令，将新的URL更新到文件.git/config；
再使用命令初始化子模块：git submodule init
最后使用命令更新子模块：git submodule update





