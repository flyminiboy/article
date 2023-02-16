## 微软Bing ChatGPT免费体验名额申请教程（保姆级）

最近ChatGPT的风头可谓是席卷全球啊，在各行各业中都激起了不小的浪花。

但是国内小伙伴比较痛苦，ChatGPT体验门槛略高，首先必须翻墙😭😭😭😭😭，另外还需要一个国外手机号进行接收验证码（当然这个可以通过接码平台操作）。如果小伙伴顺利的注册了，请一定要及时申请一个**openai_key**（免费）。这样之后再出现各种奇奇怪怪的无法访问问题，至少最为码农还是可以通过API对接来操作的。

前几天微软发布新版必应搜索引擎，采用了 ChatGPT 的 OpenAI 技术，可以用 AI 聊天问答问题，不过目前新版必应搜索引擎还没正式开放，需要申请加入。

根据微软之前的介绍，这些功能都是由 GPT 3.5 升级版本支持，所以也支持 ChatGPT 的 AI 语言模型，并表示它比 GPT 3.5 更强大，能够更好地回答带有最新信息和注释答案的搜索查询。

门槛明显下降了，那么下面我们就来介绍如何申请加入，体验新版的Bing.

### 下载Edge浏览器

**这个没啥好说的。必须下。**

**这个没啥好说的。必须下。**

**这个没啥好说的。必须下。**

### 注册账号

**没有门槛，正常注册流程即可**

**没有门槛，正常注册流程即可**

**没有门槛，正常注册流程即可**

### 重定向问题解决

但是很多小伙伴表示不懂如何加入申请，因为访问 WWW 的必应会被自动跳回到 CN 版。😂😂😂😂😂😂

这里我们需要安装一个「**Header Editor**」这个扩展插件。

#### 安装扩展插件

具体步骤看图。

* 进入扩展

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf2kSKicHdl2X1TaVW9hnr1ESkwUibJ1WZsfklAvic5SQaghU6GYovNDYpeg/0?wx_fmt=png)

或者

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf2YkbXqGgNCpP7O1hFFBz74jhIBmFejJEDGDibekZiamYp5tCpIA8KibRKA/0?wx_fmt=png)

* 搜索扩展插件

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf2y5alHgvbyJOQK6q89YlibjET9u03ecFWQkbGm3VPiaNgcbvtrsiaGHwJQ/0?wx_fmt=png)

* 获取扩展插件

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf2LCP8hAgibVRcncO85ZzcLpgfYJ6JsLmy9Th9gES5nQs3wrEBoV2Tj6w/0?wx_fmt=png)



#### 导入配置文件

将下面内容保存为一个JSON文件，随便命名。参考：edge.json

```json
{
  "request": [
    {
      "enable": true,
      "name": "bing-cn-to-www",
      "ruleType": "redirect",
      "matchType": "prefix",
      "pattern": "https://cn.bing.com",
      "exclude": "",
      "group": "bing-redirect",
      "isFunction": false,
      "action": "redirect",
      "to": "https://www.bing.com"
    }
  ],
  "sendHeader": [
    {
      "enable": true,
      "name": "bing",
      "ruleType": "modifySendHeader",
      "matchType": "regexp",
      "pattern": "^http(s?)://www\\.bing\\.com/(.*)",
      "exclude": "",
      "group": "bing-direct",
      "isFunction": false,
      "action": {
        "name": "x-forwarded-for",
        "value": "8.8.8.8"
      }
    }
  ],
  "receiveHeader": [],
  "receiveBody": []
}
```

具体步骤看图。

* 进入「**Header Editor**」扩展设置

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf21CNCGibmNJkU1JBVia1CuYkEqfVYtbn9USMMXibBaFoq3M3jc2BbaIOTg/0?wx_fmt=png)

* 选择导出和导入-》导入

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf2yOUcUvMoLcYyHrMbZag5FnDu9xXd5ptz7AOtQSic6N2fdhVauDibmiadQ/0?wx_fmt=png)

* 保存

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf2mJOM9RowFhBYP1nVdn0ChwDeLyvApFzVI2dFkHRWicVfJ3PV41swIDg/0?wx_fmt=png)

### 设置语言

打开右上角的设置 – 语言 – 选择英文，这一步根据本人验证非必须。

### 申请加入

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf2WxPeXX5GxjHj3737WN8IQtOeyU8HBwQ8iavaRtrULGiculF4rfH6FQNA/0?wx_fmt=png)

### 加速申请

点击申请加速，根据提示完成即可，微软官方软件不必有过多的担心。就是借机做了一些国内玩烂的捆绑操作。

![](https://mmbiz.qpic.cn/mmbiz_png/ibExRe3rl9wfqYKqmjmrpPs1zxsm6KYf243gk30GT79b5icZGDZEZNAbg50ZjZXpXNliabTGgCJf1AjItrShXD6sg/0?wx_fmt=png)

电脑下载一个软件做一下默认配置，手机下载一个移动端Bing，都没有门槛

### 等待，等待，等待。

收到微软那边的邮件提示申请通过，即可体验新版Bing.



