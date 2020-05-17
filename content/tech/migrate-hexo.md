---
title: 从wordpress迁移到hexo
date: 2016-03-04 14:04:12
slug: migrate-hexo
dropCap: false
---

## 为什么要用hexo

一直觉得wordpress太过臃肿，且插件太多，没有一个好的替代品。舍友推荐了静态博客，感觉挺不错的。比较火的有jekyll和hexo，正好v2ex里有人分享了一个hexo主题----**fexo**。感觉挺简洁的，于是入坑了。其实[Maupassant](https://www.haomwei.com/technology/maupassant-hexo.html)主题也不错。

## 流程记录

[hexo官方文档](https://hexo.io/zh-cn/docs/)

[fexo主题](http://forsigner.com/2016/02/23/fexo/)

本来是准备在ubuntu虚拟机里安装环境的，结果装nodejs的时候出现了非常多的问题，无奈还是选择windows。
windows直接就有一个nodejs安装包，这个是最方便的。安装完了以后，安装hexo会出现问题，google一下，再装缺少的东西就行了。然后会有两个warning，不用管，基本可以使用了。

接下来是照着文档导出_wordpress_文章，然后自己再根据需要修改一下。配置一下config.yml，然后generate一下, ，deploy即可。至此，基本一个站就出来啦。

## 问题记录

>1. deploy设置问题
>2. ignore问题

我用的是ftpsync，因为不是放在github上的，配置信息如下，具体可见官方文档。

```
deploy:
	type: ftpsync
	host: xxx
	user: xxx
	pass: xxx
	remote: /xx/xx/xx
	port: 21
	ignore: favicon.ico
	verbose: true
```

这里要注意的是remote信息，也就是路径，最好上ftp看一下，直接复制，不要自己想当然，不然很容易deploy失败的。

第二个问题就是_ignore_无效的问题，每次deploy的时候会把你那个目录地下所有的东西都删掉，例如icon。你设置了ignore，但是还是删除了，目前还没找到解决方案。我的解决档案是，把东西放在themes/fexo/source/下，这样就会把东西都一并传上去了。

总的来说，速度和美观程度很不错，关于使用七牛什么的，这是下一步的考虑方案了。

以上。

## 更新 

上次说的ftp的ignore问题，具体可参考这篇文章。[ftpsync的坑](http://cc8c.net/2015/hexo-ftpsync-bug/)

&nbsp;

## 再次更新 

不知为何，那个博客居然404了。。

废话不多说，还是解决ignore的问题，将其改成s如下格式就行了。

```
ignore: ['favicon.ico','game.html','Baidu_ife2016', 'qrcode.html']
```