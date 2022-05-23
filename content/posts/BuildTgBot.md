---
author: Runze Lee
title: 使用nodejs快速上手Telegram Bot
date: 2022-05-22
description: 使用nodejs写Telegram Bot详细教程
image: image/category_telegram_image.webp
categories: 
     - Tech/Telegram
tags:
     - 教程
---

近期在背文言文加点字的时候，我突然开始想有没有机翻文言文的网站。于是，我便发现了百度翻译的宝藏--文言文翻译。

![图1](image/build_tg_bot_1.png)

百度翻译的文言文翻译可谓全网首创，我没有再看到过哪个大厂做这个东西。不过我估计百度自己有一个名篇的库，名篇翻译还至少像样（比如说上图的出师表），如果是名气比较小的篇目就翻译得乱七八糟了。

当然，本文要讲的是写一个Telegram Bot的教程，我用Telegram Bot调用百度翻译API实现了基础的文言文翻译功能，练了练手，大家可以在[GitHub](https://github.com/Runzelee/wyw-bot)、[Telegram](https://t.me/runze_bot)上找到。

## 搭建Telegram Bot

Telegram Bot的API非常强大，可以通过API实现很多功能，比如支付、游戏、自定义键盘、群管理等等，提供很完善的交互体验。API使用GET、POST请求，调用很简单，服务端还可以使用Webhook提升体验，不用再使用愚蠢的原始长轮询 ~~（本地调试就没办法了）~~。

Webhook是指服务端反请求客户端实现实时监听的逻辑，而长轮询是指客户端隔一段时间请求一次服务端，轮环发送请求，非常耗费资源。

### 找Botfather生小Bot

如果要创建一个新的Bot，就要去找[Botfather](https://t.me/botfather)（Botfather也是一个Bot，谷歌翻译亲切地称他为“博父”），创建Bot并且拿到token。发送`/newbot`给Botfather，按照他的提示简单地创建一个Bot。

![图2](image/build_tg_bot_2.png)

最后会得到一个token，类似`123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`这样，这个token必须严格保密，因为拿到这个token的人就可以光明正大地完全控制你的Bot。

接下来，你就可以去搜索你给Bot起的用户名，向你的Bot发送消息了。然后在`https://api.telegram.org/bot123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11/getUpdates`便可看到你给bot发送的消息，这是一个基础的请求。返回的内容是JSON：
```
{
    "ok": true,
    "result": [
        {
            "update_id":***,
            "message": {
                "message_id":***,
                "from": {
                    "id":***,
                    "is_bot": false,
                    "first_name": "***",
                    "last_name": "***",
                    "username": "***",
                    "language_code": "en-us"
                },
                "chat": {
                    "id":***,
                    "first_name": "***",
                    "last_name": "***",
                    "username": "***",
                    "type": "private"
                },
                "date":***,
                "text": "***",
                "entities": [
                    {
                        "offset":*,
                        "length":*,
                        "type": "bot_command"
                    }
                ]
            }
        }
    ]
}
```

`from`、`chat`、`text`等全面体现了消息发送者、发送的内容等详细信息，`getUpdates`是专为长轮询设计的，你可以使用Python、Java等等很方便地拿到数据。除了`getUpdates`命令，还有`getMe`、`setWebhook`等等，要调用直接替换URL最后的值即可，其他命令可以看[Telegram Bot API的原始文档](https://core.telegram.org/bots/api)。

当然，这样太麻烦了，我比较懒不想调用原生API，GitHub上已经有很多现成的库简化API的调用，有现成的何乐而不为？比较著名的有[Python库](https://github.com/python-telegram-bot/python-telegram-bot)、[Node.js库](https://github.com/yagop/node-telegram-bot-api)。这篇文章主要讲Node.js库，因为我比较熟悉JavaScript以及它的异步模式 ~~，Python是外星人语言~~。

### 使用node-telegram-bot-api实现基本逻辑

有了现成的库，实现Bot基本逻辑就非常简单了，node-telegram-bot-api就是一个node module，如果你还没有装nodejs，可以参考[官方文档](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)针对不同平台安装。然后就是创建一个新项目并引入bot-api库，只需一个命令：
```
npm i node-telegram-bot-api
```
这里极力推荐vscode，编写代码和git操作都很方便，还轻量、开源。  
创建一个main.js，写入以下代码：
```
const TOKEN = 'YOUR_TELEGRAM_BOT_TOKEN';
const TelegramBot = require('node-telegram-bot-api');
const options = {
  polling: true
};
const bot = new TelegramBot(TOKEN, options);


bot.onText(/\/start/, msg => {
  bot.sendMessage(msg.chat.id, 
    "I'm a bot"
  );
});
```
`TOKEN`写入你的Bot的token，`options`是接入API的模式，可以选择轮询和webhook，先以轮询为例，因为本地调试没办法使用Webhook，后文会提到Webhook。`polling: true`即为使用轮询。`TelegramBot`提供了大量的方法，与原生API一一对应，比如下面的`onText`方法，这些方法都返回Promise，故使用回调完成相应操作。

以`onText`为例，即为监听指定消息，一旦消息匹配`/\/start/`正则表达式，就调用`sendMessage`方法。

接下来即可开启调试：
```
node main.js
```

![图3](image/build_tg_bot_3.png)

这是官方的实例，但对于Bot Command来说，这样的正则设计有很多漏洞，比如说我发送`/startyourmom`、`/yourmum/start`也会有回应。经过多次推敲，使用`/(/(\/start$)|(\/start\s+)/`比较合适，可以过滤掉上述情况，只有单独发送`/start`或者指定参数`/start yourmum`时给予回应。

另外，如果要使用Markdown语法，需要在`SendMessage`方法的第二个参数加上`{parse_mode: 'Markdown'}`，如果使用MarkdownV2（我比较推荐），则使用`{parse_mode: 'MarkdownV2'}`。下面是到目前为止的代码：
```
const TOKEN = 'YOUR_TELEGRAM_BOT_TOKEN';
const TelegramBot = require('node-telegram-bot-api');
const options = {
  polling: true
};
const bot = new TelegramBot(TOKEN, options);


bot.onText(/(\/start$)|(\/start\s+)/, msg => {
  bot.sendMessage(msg.chat.id, 
    "I'm a *bot*",
    { parse_mode: 'MarkdownV2' }
  );
});
```
这样一来，在本地环境，一个可以说话的Bot就写好了。当然一个复读机Bot只是项目的开始，可以去node-telegram-bot-api库了解完整[API文档](https://github.com/yagop/node-telegram-bot-api/blob/master/doc/api.md)，构建自己的个性化Bot。

## 部署Telegram Bot

本地完成调试之后，如果没有问题，就可以部署到自己喜欢的服务端了。你可以选择部署到免费的Web App服务商，比如说[Heroku](https://heroku.com/)、[Render](https://render.com/)等，也可以部署到自己的VPS上。这里以Heroku举例 ~~，因为我没钱租VPS~~。

首先去Heroku注册一个新账户，然后Create New App。

![图4](image/build_tg_bot_4.png)

你可以选择绑定GitHub，同步仓库创建app，也可以直接使用Heroku自己的CLI。我推荐先使用Heroku CLI推送代码，因为初始推送到生产环境时由于本地环境不同可能产生的问题还没有经过排查，等确认没有问题时再提交到GitHub仓库更合适。

但是，在推送代码之前，还有一件重要的事情，就是设置Webhook。在生产环境使用长轮询是很愚蠢的做法。

### 设置Webhook和环境变量

在main.js的开头替换成下面的代码，当然原来的代码注释即可，本地调试还需使用：

```
const TOKEN = process.env.TELEGRAM_TOKEN;
const TelegramBot = require('node-telegram-bot-api');
const options = {
  webHook: {
    port: process.env.PORT
  }
};

const url = process.env.APP_URL;
const bot = new TelegramBot(TOKEN, options);
bot.setWebHook(`${url}/bot${TOKEN}`);

//bot.on... 
```
这一段代码主要替换了两个地方，一个是`options`常量，替换成了`WebHook`。还有一个是`TOKEN`替换成了环境变量`process.env.TELEGRAM_TOKEN`。前者是为将API调用模式从轮询改为Webhook，而端口`port`，Heroku的环境变量会默认配置，没必要手动配置了。

后者是把类似`TOKEN`这样重要的参数设为环境变量读取，因为不太适合将这些密钥提交到GitHub仓库。设置环境变量很容易，可以直接去Heroku项目的Settings添加。

![图5](image/build_tg_bot_5.png)

另外，设置的Webhook地址参数`APP_URL`默认为`https://<your_proj_name>.herokuapp.com:443`，这个参数也需要在环境变量中添加。

最后，只需在package.json中添加：
```
"scripts": {
    "start": "node main.js"
}
```
这样一来，部署的前期准备工作就完成了。完成你的git的初次commit，然后就安装Heroku CLI提交代码（这里以macOS为例，具体按照[官方文档](https://devcenter.heroku.com/articles/heroku-cli)针对自己的平台安装）：
```
brew tap heroku/brew && brew install heroku
heroku login
heroku git:remote -a <your_proj_name>
git push heroku master
```
完成之后，你的项目就部署成功了，尝试向Bot发送消息，理论上应该没有问题。如果出现问题，可以调取生产环境的log：
```
heroku logs
```

### 封装环境配置

从此开始，你要从生产环境调回本地调试，或者调试完后又要再次提交的生产环境都非常地麻烦，因为环境地每次切换都伴随着替换Webhook、轮询和环境变量的重复工作。而且，本地开发环境设置的`TOKEN`等重要信息还不能一注释了之，因为要提交到GitHub仓库，导致每次都需要记下`TOKEN`。为弥补这个调试黑洞，我们必须封装环境配置来减少不必要的工作。

封装配置很简单，只需要创建两个专为环境配置的js文件，可以分别取名`prod_config.js`和`dev_config.js`，与生产环境和开发环境一一对应，内容则是创建相应环境下的bot对象：
```
//dev_config

const TOKEN = 'YOUR_TELEGRAM_BOT_TOKEN';
const TelegramBot = require('node-telegram-bot-api');
const options = {
  polling: true
};
const bot = new TelegramBot(TOKEN, options);
```
生产环境同理：

```
//prod_config

const TOKEN = process.env.TELEGRAM_TOKEN;
const TelegramBot = require('node-telegram-bot-api');
const options = {
  webHook: {
    port: process.env.PORT
  }
};
const url = process.env.APP_URL;
const bot = new TelegramBot(TOKEN, options);
bot.setWebHook(`${url}/bot${TOKEN}`);
```
然后，还需要在每个环境文件底部导出对象：
```
module.exports = bot;
```
接下来，在main.js中，在开头使用`require`引入`bot`对象即可：
```
//const bot = require('./prod_config');
const bot = require('./dev_config');
```
这样，在每一次调试完毕要切换环境时，只要注释掉一行代码就可以完成之前替换Webhook、轮询和环境变量的大量工作。当然，`dev_config.js`肯定要在`.gitignore`中添加，防止本地配置被传到GitHub仓库。

## 结语

node-telegram-bot-api是Telegram Bot界非常优秀的库，借助这样的一整套API，可以开发出前所未有的优秀作品。这是我今年重建博客以来的第一篇文章，还是以教程的口吻写的。像我这种小白也没有资格写什么教程，所以我以后会尽量多发一些自己经历的文章，多以自己的口吻来写，谢谢阅读！