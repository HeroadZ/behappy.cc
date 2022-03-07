---
title: "Google reCAPTCHA使用教程"
date: 2022-03-06T17:15:55+09:00
slug: "google-recaptcha-tutorial"
dropCap: false
tags:
  - demo教程
---

生活在互联网时代，我们会经常要跟网页打交道。即便现在手机APP也在不断的发展，Web应用依旧是最古老最常见的。在与用户进行交互的过程中，验证码有时候扮演着重要的角色，比如在用户注册，登录，提交表单的场景中，为了防止机器人浪费服务器的资源，我们需要验证在操作的是真实的用户而不是机器人。作为开发者我们当然可以从零开发一个验证码组件，不过很可能会不够完善。 [Google reCAPTCHA](https://developers.google.com/recaptcha) 是一个不错的选择，一个月可以支持一百万次免费验证。

然而在搜索了Google reCAPTCHA相关的资料[^1][^2][^3]后，发现现有的教程并不是很完善，比如：
1. 没有很详尽的前后端分离验证代码（可能因为起服务器比较麻烦
2. 几乎所有的demo代码语言都是PHP（其实就是想拿Python写一个:grin:
3. 最新的v3版本的资料很少

所以我写了一个 [Python版本的demo](https://github.com/HeroadZ/google_recaptcha_demo)，直接在本地起服务器打开网页就可以看到所有版本的效果，并且代码简单易于调试，有兴趣的可以动手操作一下以加深对Google reCaptcha的理解。下面我会结合代码和效果来介绍各个版本的使用方法。

## 注册账号和配置

首先去 [Google reCAPTCHA Admin Console](https://www.google.com/recaptcha/admin) 注册账号。
创建一个配置，demo如下：
<img src="/images/register-recaptcha.png" alt="register google recaptcha" width="400"/>
需要注意的有两个地方：

1. 一个是选择想要使用的reCAPTCHA类型，目前有3种。显式复选框的v2版本，隐藏式的v2版本以及最新的v3版本。所以对于不同的版本，我们需要创建不同的配置。这三种类型的区别可以见下表：
|                | 显式v2 |  隐式v2  |   v3   |
|:----------------|:------|:-------|:-----|
| 是否显示验证框   | :o:    |  :x:    |    :x: |
| 何时验证        |   点复选框  |   提交   |    提交   |
| 验证返回数据        |    是否成功    |    是否成功      | 用户可信度(0-1) |

2. 还有一个是域名，我们这边只是本地环境，所以用了`127.0.0.1`。如果需要支持线上环境，可以添加其他域名。

配置完可以拿到我们后续需要用到的一对密钥，一个是添加在html页面里面的public key，一个是后端验证时需要的secret key。

## 显式v2版本
<img src="/images/recaptcha_v2_checkbox.png" alt="rrecaptcha v2 checkbox" width="500"/>
从上面的图片不难看出html部分的代码很简单：

```html
<h1>google recapatcha v2 checkbox demo</h1>
<form id="form" method="POST">
    <div id="g-recaptcha"></div>
    <p id="capatcha-message"></p>
    <button id="send" type="submit">submit</button>
</form>
```

接下来根据 [reCAPTCHA文档](https://developers.google.com/recaptcha/docs/display) 我们需要引入js用来渲染验证复选框。js部分代码为:

```html
<script>
    const public_key = 'public key'
    var widgetId
    var onloadCallback = function () {
        // 渲染验证复选框，第一个参数为DOM id，第二个参数为配置
        widgetId = grecaptcha.render('g-recaptcha', {
            'sitekey': public_key,
            'theme': 'light',  //主题颜色，有light与dark两个值可选
            'size': 'normal',  //尺寸规则，有normal, compact和invisible三个值可选
            'callback': verifyCallback, //验证成功回调
            //还有 expired-callback 和 error-callback 两个回调函数，一般不用管默认就好
        })
    }

    // 本地验证成功后的回调函数
    const verifyCallback = function (token) {
        document.getElementById('capatcha-message').textContent = "verification success locally";
    }
</script>
<script src="https://www.google.com/recaptcha/api.js?onload=onloadCallback&render=explicit" async defer></script>
```
### url参数说明
除了渲染复选框可以指定参数外（代码里的注释部分），引用reCAPTCHA js的时候有3个可选参数。
- onload：加载完依赖项时的callback函数
- render：是否显式加载组件，默认值为onload，表示自动加载，也就是默认找到第一个class为g-recaptcha的标签来加载组件。例子中我们设置的值为explicit，意思是不启用自动加载，而是根据我们提供的DOM id进行加载。
- hl：语言，不设置的话自动检测浏览器语言作为标准，建议不设置。

### 验证流程
<img src="/images/recaptcha_verify_process.png" alt="recaptcha verify process" width="600"/>
所以我们上面的代码就是当加载完js之后渲染验证复选框，当本地验证成功后显示验证成功的文字。这里的验证仅仅是本地验证，下一步呢就是服务端验证。具体的流程是，当本地验证成功后会产生一个token，我们需要把token发送到后端，后端把这个token发送到google的服务器，然后google服务器会返回一个验证结果，我们可以通过这个验证结果来判断用户是否通过验证。



将token发送到后端这部分工作我们利用js完成：

```js
document.getElementById("form").onsubmit = function (e) {
    // 阻止submit事件冒泡
    e.preventDefault();

    // 如果response(也就是token)不为空，代表用户本地验证成功
    if (grecaptcha.getResponse(widgetId).length > 0) {

        // 将token发送给后端
        var xhr = new XMLHttpRequest();
        xhr.open("POST", "http://127.0.0.1:8000", true);
        xhr.setRequestHeader('Content-Type', 'application/json');
        xhr.send(JSON.stringify({
            "token": grecaptcha.getResponse(widgetId),
            "version": "v2_checkbox", // 为了server端区分版本使用，不是必须
        }))

        // 接受后端返回的google服务器验证结果
        xhr.onreadystatechange = function () {
            if (this.readyState == xhr.DONE && this.status == 200) {
                var response = JSON.parse(xhr.responseText);
                console.log(response)
                document.getElementById('capatcha-message').textContent = `verify ${response['success'] ? 'success' : 'fail'} from google`
            }
        }

        // 因为token只能被验证一次，所以一旦送去google验证后就得reset
        grecaptcha.reset(widgetId);
    } else {
        // 用户没有通过本地验证，请求用户先验证
        document.getElementById('capatcha-message').textContent = "please verify capatcha"
    }
}
```

服务端代码比较简单，python的话直接用[requests库](https://docs.python-requests.org/en/latest/)发post请求就行。
```py
r = requests.post('https://www.google.com/recaptcha/api/siteverify', data={
    'secret': 'secret key', # 填写你的secret key
    'response': 'token' # 填写前端发来的token
})
```
### demo运行效果gif

demo运行效果可以在下面的gif看到：
<img src="/images/recaptcha_v2_checkbox.gif" alt="recaptcha v2 checkbox gif" width="500"/>


上面一趟完整流程走下来，服务端会收到前端传来的post请求和google返回来的验证结果，都可以在console里打印出来看，为了方便观看，这里的token改了一下，可以看到验证结果为成功。

```sh
post data: {"token":"xxxxxx","version":"v2_checkbox"}
127.0.0.1 - - [07/Mar/2022 00:40:40] "POST / HTTP/1.1" 200 -
verfication result from google: {'success': True, 'challenge_ts': '2022-03-06T15:40:11Z', 'hostname': '127.0.0.1'}
```

所以显式v2的版本仅仅本地验证成功是不行的，因为如果有人用Postman做了一个假post请求，token随便设置成`"123"`的话就能绕过前端本地验证。

## 隐式v2版本

之前的显示v2版本有两个缺点：
1. 用户在提交之前多了手动验证这一步，降低了用户体验。
2. 本地验证因为可以随便绕过显得有点多余，是否可以将两步验证合并成一步。

为了改进上面的缺点，隐式v2版本选择在用户点击提交按钮的时候本地验证+google验证。这样做的好处是对于真实用户，图片验证可能不会被触发，所以用户感觉不到验证这一步体验不会下降，而机器人在提交的时候可能会被要求验证。

相比显式v2版本，代码改动主要有3个地方，详情浏览[github demo](https://github.com/HeroadZ/google_recaptcha_demo):
1. 渲染验证码的时候，size改成invisible。
2. 把后端验证代码合并到verifyCallback里，之前本地验证和后端验证是分开的。
3. submit监听事件函数中运行`grecaptcha.execute(widgetId)`来触发验证。

### demo运行效果gif

<img src="/images/recaptcha_v2_invisible.gif" alt="recaptcha v2 invisible gif" width="500"/>

## v3版本

其实上面的隐式v2版本已经是非常不错了，但是还有一个问题，那就是验证通过与否全由google api来判断，用户没有一丁点决策权，过于霸道了。所以v3版本的改进是，对于不同组件(action)，返回一个0-1范围内的用户真实度score，用户可以根据score来决定验证是否通过，默认阈值为0.5。与此同时，如下图所示我们也可以得到我们网站各个组件的验证通过比率和score分布，图片来自[谷歌官网v3视频](https://youtu.be/tbvxFW4UJdU)。

<div style="display:flex;">
<img src="/images/grecapcha_v3_top10.png" alt="recaptcha v3 top 10 actions" width="400"/>
<img src="/images/grecaptcha_score_distribution.png" alt="recaptcha v3 score distribution" width="300"/>
</div>

假设现在网站有两个组件需要验证，一个是用户登录，一个是评论。经过1个月的运行，发现用户登录组件的score分布中频率最高为0.7， 评论组件的频率最高score为0.4。那么我们或许可以根据判断来适当提高登录组件的score判定阈值来抵挡掉更多机器人的请求，或许可以适当降低评论组件的判定阈值来增加评论件数。

有两个地方需要代码改动，详情浏览[github demo](https://github.com/HeroadZ/google_recaptcha_demo)：
1. 引入api.js的时候，render选项改成publick key。
2. 渲染组件的代码精简了，在onsubmit监听函数中直接ready初始化, execute执行验证。

### demo运行效果gif

<img src="/images/recaptcha_v3.gif" alt="recaptcha v3 gif" width="500"/>

可以看到对于我们定义的submit action，这次验证的分数为0.9。google api验证返回的内容是
```json
{
  "success": true,
  "score": 0.9,
  "action": "submit",
  "challenge_ts": "2022-03-07T04:13:09Z",
  "hostname": "127.0.0.1",
}
```

## 总结
Google reCAPTCHA从v1版的文字验证，到v2版的图片显式/隐式验证，再到v3版的score验证，的确是解决了之前存在的问题，变得越来越好用了。Google内部靠什么来进行风险分析不会公开出来，可能会用到你的IP请求数，浏览器指纹，谷歌账号cookies之类的。当然Google reCAPTCHA不是万能的，肯定有方法可以hack，但是肯定会比较复杂，能够提高攻击者的攻击难度和成本我觉得这就足够了。以上。


<br />
<br />
<p style="text-align: center;">如果本文对您有帮助，欢迎打赏。</p>
<img src="/images/qr-wechat.png" alt="赞赏码" width="300"/>

[^1]: [如何使用reCaptcha（2.0版本）来做网站验证码](https://blog.csdn.net/tsyccnh/article/details/50978611)
[^2]: [google recaptcha 谷歌人机身份验证超详细使用教程，前端/后端集成说明](https://www.cnblogs.com/echolun/p/12436226.html)
[^3]: [Google reCAPTCHA の使い方（v2/v3）](https://www.webdesignleaves.com/pr/plugins/google_recaptcha.php)
