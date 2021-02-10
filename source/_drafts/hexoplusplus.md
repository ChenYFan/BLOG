title: HexoPlusPlus-从一个妄想到现实
author: CYF
tags:
  - Hexo
  - 集成部署
categories:
  - 好方法
date: 2021-2-5 15:40
index_img: https://cdn.jsdelivr.net/gh/ChenYFan/CDN@master/img/hpp_upload/1612828904000.jpg
banner_img: https://cdn.jsdelivr.net/gh/ChenYFan/CDN@master/img/hpp_upload/1612828904000.jpg
---

我一直都习惯在线写作，但因为口袋里没钱，不能买服务器用动态博客，使用Hexo，即使实现了集成部署，想要在github上直接书写，尤其是出门在外有所灵感，国内手机登陆github真的是极其糟糕的体验。博客本就是碎片化写作和高质量文章发布处，使用hexo却使我无法发挥博客的用处。

先前，我曾使用白嫖的Euserv搭建的Typecho，也是用过wordpress.com白嫖的wordpress，，虽然但两个都不符合我对速度和可用性的追求，一个连CloudFlare能不能连上都是问题，另一个中国支持贼差【虽然可以用万能Worker可以替换加解决,但是就是不想用啊】。免空的选择又难以择手![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.11/194.jpg)，弄来弄去还是用回Hexo。

但是Hexo就是有一点不爽，每次使用的时候就必须要在本地进行构建静态网页，然后上传到GithubPage。后来实现了集成部署【没想到折腾了很长时间的集成部署最后用到这里了】，方便了不少，直接在Github上面改源代码。但相较于Typecho和Wordpress，没有后台的写作总感觉十分不方便![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.8/5896e9710dfd5.jpg)，每次更改源代码都要上Github![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.8/5896ece2ab57a.jpg)，在国内这种大环境下总是不方便的。

2020年最后一个月，我总是在想如何解决这个问题，我的要求很简单，能弄个在线书写环境就好了。

由于我的文件是存储在Github上，于是我第一个先去Github文档查找相关资料，果不其然，Github的API能够上传、删除、下载【废话】、列表文件，并且能通过base64上传，直接免去了手写头的问题.关于调用限制，没鉴权时每个ip每小时只有**60次**，但一旦鉴权每个用户每小时就有**5000次**。这些api完全能够支撑起一个在线写作的环境,<https://developer.github.com/v3/guides/getting-started/>更是详细讲解并提供了数个例子。

# 原理 - GithubAPI

譬如罢，上传一个文件，首先你要鉴权，在header中写入：

```url
content-type: "application/json;charset=UTF-8"
Authorization: "token  ${hpp_githubimagetoken}"
```

> Anyone,你也可以在url后面加上`?access_token=`传参，但是这样不安全，Github官方也是提示将在明年彻底禁用传参鉴权

但是记得GithubAPI不允许空User-Agent，所以你还得在header中加入UA：

```url
user-agent: "GoogleChrome",
```

OK这么一搞鉴权这一块就完毕了，接下来，我们要搞基本功能

Github更改一个文件的url是一样的，为了方便接下来的书写和表达，我们统一将以下url称为RESTURL：

```url
https://api.github.com/repos/${Github用户名}/${Github仓库名字}/contents/${Github文件路径}/${Github文件名}?ref=${Github分支}
```

## 拉取信息

默认情况下，直接`GET``RESTURL`就能获取该文件/文件夹的信息,例如获取我`AVorBV.md`源文件,那么`RESTURL`如下:

```
https://api.github.com/repos/ChenYFan/blog/contents/source/_posts/AVorBV.md?ref=master
```

直接`GET`[我的是公开仓库,不需要鉴权就能获取],得到数据如下:

```json
{
"name": "AVorBV.md",
"path": "source/_posts/AVorBV.md",
"sha": "a0bd826f999a9bb73ac56251415f9e57199600a7",
"size": 15742,
"url": "https://api.github.com/repos/ChenYFan/blog/contents/source/_posts/AVorBV.md?ref=master",
"html_url": "https://github.com/ChenYFan/blog/blob/master/source/_posts/AVorBV.md",
"git_url": "https://api.github.com/repos/ChenYFan/blog/git/blobs/a0bd826f999a9bb73ac56251415f9e57199600a7",
"download_url": "https://raw.githubusercontent.com/ChenYFan/blog/master/source/_posts/AVorBV.md",
"type": "file",
"content": "dGl0bGU6IEFWP0JWIQphdX...",
"encoding": "base64",
"_links": {
"self": "https://api.github.com/repos/ChenYFan/blog/contents/source/_posts/AVorBV.md?ref=master",
"git": "https://api.github.com/repos/ChenYFan/blog/git/blobs/a0bd826f999a9bb73ac56251415f9e57199600a7",
"html": "https://github.com/ChenYFan/blog/blob/master/source/_posts/AVorBV.md"
}
}
```

这样子,我们只要提取json中的sha,就能知道到hash,进而进行修改.
但这样子有个非常尴尬的一点,单文件获取会把`content`一口气拿过来
例如下面的`RESTURL`

```url
https://api.github.com/repos/ChenYFan/CDN/contents/img/hpp_upload/1612843011000.jpg?ref=master
```

你获取的时候会发现返回了这个:

```json
{
"message": "This API returns blobs up to 1 MB in size. The requested blob is too large to fetch via the API, but you can use the Git Data API to request blobs up to 100 MB in size.",
"errors": [
{
"resource": "Blob",
"field": "data",
"code": "too_large"
}
],
"documentation_url": "https://docs.github.com/rest/reference/repos#get-repository-content"
}
```

很显然,直接用GithubAPI不能获取单个文件的hash值

那怎么办？

答：列表获取

我们把之前的`RESTURL`去掉小尾巴,变成这样:

```url
https://api.github.com/repos/ChenYFan/CDN/contents/img/hpp_upload?ref=master
```

这样就能获取这个目录下整个列表,然后用json循环查找遍历name,再通过name拉hash即可.

只是这样查询时间会略微变长.

## 新建

如果是新建,body中这么写

```json
{
    branch: ${上传的分支},
    message: ${上传的信息},
    content: ${base64过的文件}, 
    sha: ""
}
```

接着使用`PUT`形式访问RESTURL

创建成功后状态码应该返回：

```status
201 Created
```

## 更新

body与新建类似，但是首先你要通过拉取信息获取该文件sha值.

```json
{
    branch: ${上传的分支},
    message: ${上传的信息},
    content: ${base64过的文件}, 
    sha: "${此文件hash}"
}
```

接着使用`PUT`形式访问`RESTURL`

更新成功后状态码应该返回：

```status
200 OK
```

## 删除

相对来说,删除就更简单了

```json
{
    branch: ${删除文件的分支},
    message: ${删除的信息},
    sha: "${此文件hash}"
}
```

hash这一步逃不掉,用`DELETE`形式访问`RESTURL`,返回`200`说明删除成功

# 原理 - CloudFlareWorkers

之前看过[Laziji-VBlog](https://github.com/GitHub-Laziji/VBlog)项目,这个项目新颖的一点是将文章发布在gists,然后用户通过api访问获取.

但这样有两个致命问题:

1.API没鉴权，每小时单个ip只能访问60次，一开就爆
2.在国内受干扰，不稳定

并且什么迁入迁出麻烦、token容易忘记等等问题

最最最早版本中,我是打算纯静态实现文章编辑和更改的，但很快我就遇到了和VBlog一样的缺陷，这逼使我切换了平台。

好诶，既然直连效果那么差，我们就选择中继。利用服务器中继我们首先排除【用Hexo基本就是贪无服务器】。目前比较流行的无服务器平台有Heroku、CloudFlareWorker和Vercel，Heroku支持了多种服务器语言，CFWorker基于GoogleV8，因为JSProxy在国内意外走红，Vercel在国内拥有较好的运营商线路。

我们第一个排除heroku，冷启动唤醒需要10s，并且无法绑定域名【这里其实也可用worker反代（bushi】。目光看向worker和vercel，又有一个新问题出来，自定义配置存哪？![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.8/5896e6ec1d528.jpg)

存变量里当然是个好主意，但是很难修改。外部存储也不是什么大问题，mongodb、firebase、~~Leancloud~~都可以上手，但我个人终究不喜欢为了查询而发送子请求。![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.11/194.jpg)

由于我是OIer【虽然很差】，习惯使用C++的逻辑，因为JS的逻辑和C++其实差不多，所以我更倾向用WorkerJS书写。

非常赞的是，去年11月，[CloudFlare官方宣布KV在一定额度内免费](https://blog.cloudflare.com/workers-kv-free-tier/)，并且免费额度喜人：

```
存：1GB大小
读：10W次/天【注：这里和Worker免费版本调用次数相同】
写：1k次/天
删：1k次/天
列：1k次/天
单个限额：25MB
```

并且worker里面使用KV函数异常简单，绑定KVNAME后：

```js
async function FUNCNAME(){
await KVNAME.get(INDEX) //读
await KVNAME.put(INDEX,VALUE) //写
await KVNAME.delete(INDEX) //删
}
```

按照[官方文档](https://developers.cloudflare.com/workers/learning/how-kv-works)的说法，实际读取与读取静态页面差不多，我写了个简单测试函数，根据时间戳判断，单次读取只需要不超过2ms。

并且worker有非常赞的fetch函数，无痛自定义header，拉取后端无压力。

好，那么就开始吧。

# 实现 - 迈出的第一步

首先你要绑定个监听器：

```js
addEventListener("fetch", event => {
  event.respondWith(handleRequest(event.request))
})
```

由于`fetch`只能在`async`函数执行,于是我们写个`async`:

```js
async function handleRequest(request) {
return new Response()
}
```

可以，这样我们就简单实现了一个无服务器函数![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.8/5c53d1904dcb2.gif)

接下来的函数就应该在async这个主函数写。

然后是最基本的fetch，fetch应该说是worker里最特色的函数了。

如果直接返回，那么就不用加`await`,因为在`async`里面返回了一个`await`

```js
return fetch('https://api.github.com/repos/ChenYFan/blog/contents/source/_posts')
```

如果要拉回来做运算，那么要加`await`

```js
const res = await fetch('https://api.github.com/repos/ChenYFan/blog/contents/source/_posts')
```

CFWorker能用`.text()`函数和`.json()`函数处理返回的内容：

```js
const first_name = await JSON.parse(await(await fetch('https://api.github.com/repos/ChenYFan/blog/contents/source/_posts')).text())[0]["name"]
return new Response(first_name)
```

这个其实等价下面的：

```js
const first_name = (await(await fetch('https://api.github.com/repos/ChenYFan/blog/contents/source/_posts')).json())[0]["name"]
return new Response(first_name)
```

当然显然是下面的好写,但我习惯测试方便都用上面的![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.8/stick_18.png)

我们也可以通过自定义方式来自定义header完成鉴权和UA设置:

```js
const getinit = {
          method: "GET",
          headers: {
            "content-type": "application/json;charset=UTF-8",
            "user-agent": `${USERAGENT}`,
            "Authorization": `token ${TOKEN}`
          },
}
const first_name = await JSON.parse(await(await fetch('https://api.github.com/repos/ChenYFan/blog/contents/source/_posts',getinit)).text())[0]["name"]
return new Response(first_name)
```

那么接下来就很简单了。

# 实现 - 面板的设计

Worker支持返回数据的设置，我们可以通过修改`content-type`达到返回页面的效果,并且可以通过JS奇妙的语法完成PHP难以做到的事情。

首先先定义一个网页：

```js
const re_html =  `<h1>Hello,World!</h1>`
```

【先咕咕咕，省得忘记这篇文章了![](https://cdn.jsdelivr.net/npm/chenyfan-oss@1.1.8/5896e9710dfd5.jpg)】