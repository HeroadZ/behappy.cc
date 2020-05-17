---
title: 毕业设计——基于Ruff的简单物联网
date: 2017-07-22 09:07:48
slug: Ruff-graduation-project
dropCap: false

---

# 1. 背景
## 1.1 为什么用Ruff

这次介绍我的毕业设计，这是个非常简单的东西。所有的源码在**[Github](https://github.com/HeroadZ/Ruff-graduation-project)**上，还包含了配置信息。

因为接触了好长时间的前端，然后本身是EE专业，就是搞单片机这种东西嘛，所以对js能否搞硬件一直保有期待。正好快毕业了，老师呢，也说让我自主选题。于是在网上搜索好久，先是舍友推荐了**ESP8266**。这是一块自带wifi的板子，毫无疑问，对于长期只能玩51板子的我有巨大吸引力！都能上网了还有什么不能干！然后又是一顿搜索，找到了集成度更高的**[nodemcu](http://www.nodemcu.com/index_cn.html)**，哇，这货就是我心中想要的。Nodejs编写网络应用，然后Lua操作硬件，可惜的是学习Lua也要一定的成本，不过这个很便宜，可以说是非常好的物联网开发板了。

然而我还是找到了**[Ruff](https://ruff.io/zh-cn/)**。这是一家专业的公司，有着成熟的开发方式和套件。底层SDK给你了，规范的常用的硬件模块给你了，甚至连软件封装都给你封装好了，你只需要有想法就行，剩下的几行代码足以。而且还有戴尔售后般的服务，缺点就是比较贵，板子加上官方硬件模块需要**300**块，不过老师可以报销，加上老师也挺感兴趣的，最终就决定拿Ruff来开发啦。

## 1.2 Ruff的开发方式

我们看一下Ruff的开发方式。

![Ruff之helloworld](/images/Ruff-turnOn-led.png)

真的，这种开发方式看到不心动不是人了！当板子初始化好以后，也就是ready以后，给button绑定一个侦听事件，如果按下，灯就点亮。这完全就跟网络端的前端开发方式一摸一样！传统前端也是灯页面加载好以后运行js代码的，也是基于事件驱动型的。而且这个还是JQuery语法，写起来倍儿爽！

**汇编**是这样的：
```
ORG 0000H		  ;从最开始处执行
clr P1.4           ;开led灯总使能
mov P0,#11111101B  ;修改这里,不同的发光管会亮,8个2进制位代表8个灯,0为亮,1为灭
sjmp $             ;死循环,目的是让程序一直停留在这里
end
```
是不是很烦，而且是从给引脚赋值，真的是蛋疼啊，完全没有阅读性可言。

**C语言**是这样的：
```c
sbit  led1=P1^0;
```

前面的一堆引入就不放进去了，同样是对引脚操作，东西一多，代码看起来真费劲。

所以看到js的开发方式，的确很感动。我很欣赏Ruff创始人的观点：

>嵌入式开发门槛太高，很大的原因在于抽象层次做的不够。就算用JavaScript语言去开发嵌入式端，**如果还是对pin脚，对GPIO操作一个变频来达到目标，那么就还是用原始的方法在做开发**。如今的应用开发，要么就是不需要写代码，要么就是能凭着自己的想法下意识地构思出代码。所以说，语言并不是最重要的，重要的是对**硬件的抽象**。

# 2. 开发

## 2.1 我想要做什么？

首先我要明白，我想做什么，比如说最简单的，拿手机远程控制Ruff开发板，来点亮Ruff上的一盏灯。那么我要解决的问题是——**手机如何与Ruff开发板通信**？

其实分为两个部分，第一个手机端需要有个发送信息的程序，Ruff端需要有个接收信息并处理的程序。这样一看就是很简单的**C/S（客户端/服务器）**模式，Ruff开发板就是服务器，手机就是客户端。由于我们的命令可能就是一个true或false，不需要实时，所以用最简单的get请求就可以了，连post都不需要。所以大致流程如下，不清楚的可以去恶补一下计算机网络知识：

![Ruff通信之C/S模型](/images/Ruff-CS-model.png)

那么其实这里就默认客户端需要知道服务器端的ip地址，其实这里Ruff提供了两种连接方式。第一种方式是**AP模式**，也就是手机直接连接到Ruff开发板的wifi中，没错，Ruff也是自带wifi的。还有一种是**局域网模式**，也就是手机和Ruff开发板都连接到一个局域网环境下（比如路由器等）。

**AP模式**的好处是，这种情况下，Ruff开发板的ip相对于手机是**固定不变**的，为`192.168.78.1`。只是Ruff的网不是特别稳定，而且手机连上了Ruff的网，不能连接到外网。不过现在Ruff出来了一个**内网渗透模式**，也就是只要Ruff连接到外网，并开启这个模式，手机也能上网了。

**局域网模式**的好处是，两者互不干扰，而且路由器相对稳定些。只是这时候Ruff开发板相对于手机的**ip地址会变**，因为每次Ruff连上局域网，可能会给它分配一个新ip。但是可以用**绑定MAC地址**的方式，使得Ruff开发板的ip固定不变。

由于我没有路由器，所以暂时用AP模式进行测试，注意每次开启Ruff的初始化时间比较长，而且Ruff连接到外网以后，会断掉wifi，过一段时间再重新开启。

## 2.2 客户端程序

然后可以开始写手机端的程序了，怎么搞？一开始还想着搞个React Native，写个原生APP把，后来看了官方文档，放弃了。本来就不会React，还要懂各种安卓开发技巧。于是最终就简单写了个移动端页面。flex布局+jQuery就行了，图标用iconfont的。

下面放上我毕业设计的最终图，由于Ruff开发板答辩完以后被收走了，我也懒得再重新写页面了。

![Ruff手机端主页面](/images/Ruff-phone.png)

如上所示，我最上面有3个功能模块，放后面讲，然后中间的ip修改是方便局域网时，可以修改发送地址的。其实就是把ip地址放在**[localStorage](https://developer.mozilla.org/en/docs/Web/API/Window/localStorage)**里面，每个页面都可以共享，然后有需要就改掉。注意这里需要**正则验证**一下输入，用`/^([0-9]{1,3}\.){3}[0-9]{1,3}$/.test(ipInput.val())`就可以了。然后下面的修改颜色模块，其实就是我所说的点亮一盏灯的功能了。因为这边用的是RGB三色灯，是基于PWM波。根据简单的三原色原理，我们直到**当R,G,B都是255的时候，就是白光；R,G,B都是0的时候，就灭掉**。

然后修改页面的代码大致如下，使用ES6写的，不过就语法变了下：
```javascript
$.get(localStorage.getItem('ruffUrl') 
 + 'colorChange?color=' 
 + encodeURIComponent(colorCode), (data, status) => {
 	if(status === 'success') {
 		if(data.colorChange === true) {
 			window.alert("修改成功！");
 		} else {
 			window.alert("修改失败!");
 		}
 	} else {
		 window.alert("修改失败!");
	}
});
```
可以看到，我们把颜色参数添加到了链接后面，注意这边需要编码，因为`#`是特殊字符。所以最后这个网址可能是这样的，6318是监听的端口号：
```
http://192.168.78.1:6318/colorChange?color=%23ffffff
```

## 2.3 服务器端程序

那么现在客户端可以发送请求了，服务器端必须监听这个请求并做出相应的处理。幸运地是，Node.js有相应的http库，但是所有的发送，接收，监听，回掉都要自己慢慢写，而Ruff替我们封装了一个库**[home](https://github.com/vilic/ruff-home)**，我们可以很简单地处理请求，但是需要注意的是，为了解决跨域问题，我们需要在response的header上加上允许跨域。所以最终的部分代码可能是这样的：

```js
var Server = require('home').Server;
var server = new Server();
  

server.listen(6318); //监听端口
server.get('/colorChange', function (req, res) {
    res.setHeader('Access-Control-Allow-Origin', '*'); //允许跨域
    var urlObj = url.parse(req.url);
    var queryObj = queryString.parse(urlObj.query);
    var color = queryObj.color.toLowerCase();  //颜色参数
    var sColorChange = [];
    for (var i = 1; i < 7; i += 2) {
        sColorChange.push(parseInt("0x" + color.slice(i, i + 2))); //转换成特定格式
    }
    led.setRGB(sColorChange); //设置颜色
    console.log('color Change: ' + color);
    return {
        colorChange: true //返回数据以便客户端确认
    };
});

```

# 3. 其他功能
## 3.1 自适应灯光

说白了就是led灯的亮度根据当前环境亮度（通过光照传感器获得）作出改变。需要注意的是，这里有个难点，因为led灯是RGB标准的，而无法直观改变亮暗度。所以我采用了**[HSL标准](https://zh.wikipedia.org/wiki/HSL%E5%92%8CHSV%E8%89%B2%E5%BD%A9%E7%A9%BA%E9%97%B4)**，这里L表示亮度，然后色彩只保存在H和S里。至于这个转换的代码，直接从**[stackoverflow](https://stackoverflow.com/questions/2353211/hsl-to-rgb-color-conversion)**上复制了，虽说也不难，但省事。

## 3.2 夜间睡眠模式

也就是，到了晚上（光照传感器确定），睡觉的人起床（声音传感器听到声音），led灯渐渐变亮。这里的难点是js的回掉太多了，简直回掉深渊，各种绑定取消。其实也没这么烦的，但是我想做一个类似按键防抖的功能，因为有时候会有比如说人经过会遮挡光照传感器等特殊情况。还好有人设计了**promise**这个东东，简单又使用。**张鑫旭**写了[一篇文章](http://www.zhangxinxu.com/wordpress/2014/02/es6-javascript-promise-%E6%84%9F%E6%80%A7%E8%AE%A4%E7%9F%A5/)来介绍这个概念，Ruff官方也封装了[一个专门的库](https://github.com/vilic/ruff-promise)来供我们使用。

所以一个简单的确认到晚上的功能是这样的：

```js
var nightFlag = false;

var getIlluminancePromise = new Promise(function (resolve) { //建立Promise对象
    lightSensor.getIlluminance(function (error, value) {
        resolve(value);
    });
});

getIlluminancePromise
    .then(function (illuminance) {
        if (illuminance <= nightLightValue) {
            if (nightFlag) {
                return false;
            } else {
                setTimeout(function () {
                    lightSensor.getIlluminance(function (error, value) {
                        if (value <= nightLightValue) {
                            console.log('night confirm after 3s.'); //确认晚上
                            nightFlag = true;

                            soundSensor.enable(); //打开声音传感器
                            soundSensor.once('sound', sleepLight.ledTurnOn);
                        }
                    });
                }, 3000);
            }
        } else {
            nightFlag = false;
        }
    })
```

不过声音传感器的防抖因为设计原因，我暂时还没什么好想法。

## 3.3 监控模式

In a nutshell，就是人体红外传感器检测到人，向微信发送消息。这里的难点是Ruff开发板向微信发送消息，我这里借用了**[Server酱](http://sc.ftqq.com)**。也就是说我在这里绑定我的微信，得到一个key，然后我带着这个key发送get请求给Server酱，她会自动将消息转发给我的微信，详细地可以看官方说明。

# 4. 总结

最后的实物图：
![Ruff实物图](/images/Ruff-hardware.jpg)

连接图：
![Ruff连接图](/images/Ruff-connection.png)

需要注意的是：
> 1. LCD必须接**5V**的VCC
> 2. 连接图可以自己调，改到舒服的端口
> 3. 插线很烦，最后插完不要拔了

其实还挺有趣的，我没有用到Ruff的测试环境，没有自己写驱动包，因为没有用其他模块，尝试了下ES6和webpack，最近了解到写sass可以用csscombine来调节顺序什么的。所有的东西我只学了个皮毛，所以做出来的东西也只能是皮毛。不过毕业设计能做自己感兴趣的东西还是挺好玩的。

以上。

