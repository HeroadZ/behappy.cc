---
title: pandaTV竹子Top20教程
date: 2016-08-09 09:44:44
slug: python-panda-rank
dropCap: false
---

这次我会带来一个pandaTV竹子Top20的教程，从零开始。
# 1. 背景交代
## 1.1 工具和环境：
>
1. python库：requests、pymongo等
2. 开发环境：python3.5、windows10
3. 数据库：mongodb
4. 数据可视化工具echarts.js
5. 抓包分析工具fiddler
6. 一个虚拟主机或者github pages

## 1.2 教程整体思路：
> 1. 首先我们要抓取数据（主播id和竹子数），那么我们就要抓包分析。
> 2. 当我们知道从哪里可以获取数据后，就要写爬虫。写爬虫的时候要考虑以下问题，比如并发，增量等。
> 3. 当我们抓取到所需数据时，我们需要保存数据，我们使用了数据库（其实是杀鸡用牛刀了）。
> 4. 现在我们要根据需求来提取分析数据了，然后返回我们需要的数据。
> 5. 得到所需数据后，我们要展示这些数据，要使用一些数据可视化工具。
> 6. 写好展示页面后，我们要将它部署到网站上去，会讲到一些方法。

# 2. 教程正文
## 2.1 抓包分析
我们先分析一下需求，我们希望得到的数据：
> 1. 因为pandaTV房间号是不固定的，我们要先获取热门直播里面的前20个主播的房间号。
> 2. 得到房间号以后，我们要得到每个房间号里的主播id和竹子数量。

如果不想抓包分析，那可以直接使用selenium，但是selenium是万不得已才会用的，我第一次写的版本就是selenium版的，运行速度很慢。因为它的原理就是模拟人控制浏览器，要一个一个点开网页。  

### 2.1.1 网页抓包
那么我们接下来开始抓包分析，pandaTV使用了ajax，我们使用开发者工具来抓包分析一下。放上[chrome开发者工具教程](https://developer.chrome.com/devtools)

我们打开F12，点到Network标签，把preserve log钩打上，下面的选择XHR，开始抓包分析。我们先输入网址 www.panda.tv，然后点全部，然后随便选个房间进去，此处以我的房间为例。

![pandaTV抓包](/images/panda-F12.jpg)
这里我直接给出需要的地址，即第4步的地址，其他的地址暂时不需要。那么网址格式大概`"http://www.panda.tv/api_room_v2?roomid=" + 房间号`, 比如 `http://www.panda.tv/api_room_v2?roomid=473005`
我们直接访问这个网址，得到json格式的数据，里面有我们需要的主播id和竹子数。

### 2.1.2 fiddler抓包
那我们得到了输入房间号返回数据的地址，接下来我们要获取热门直播的地址，那网页版是无法获取的，我们需要抓取Android客户端的api，这里用到的工具是fiddler，给出[fiddler教程](http://www.trinea.cn/android/android-network-sniffer/)

为了节省时间，我直接给出我们需要的地址。pageno是页数，pagenum是每页多少人，同样得到json格式数据。
<code>
	http://api.m.panda.tv/ajax_live_lists?pageno=1&pagenum=20&status=2&order=person_num
</code>

还有一个Github上的非官方API，也挺不错的。[pandatvAPI](https://github.com/MatteO-Matic/pandatvAPI)

## 2.2 写爬虫
### 2.2.1 房间号列表
我们根据Android客户端抓取到的url获取房间号列表
```python
import requests

#得到推荐前20的房间号
def getTop20List():
	top20List = []
	url = "http://api.m.panda.tv/ajax_live_lists?pageno=1&pagenum=20&status=2&order=person_num"
	info  = ((requests.get(url)).json())["data"]["items"]
	[top20List.append(data_id["id"]) for data_id in info]
	return top20List
```

那么我们再分析一下，因为这是当前的热门直播前20，主播的作息时间是不固定的，所以我们涵盖不了所有的主播。那么我们要实现增量抓取，即每次爬取的房间号要保存下来，这样反复几次，可以涵盖几乎所有的热门主播。这里我们使用pickle，这是一个很方便的库，方便的保存读取房间号列表，然后实现增量。
```python
import pickle, os.path

#记录爬取的房间号
def recordRoomList(roomNumberList):
	with open('roomList.pickle', 'wb') as f:
		pickle.dump(roomNumberList, f)

#读取文件中的房间号
def readRoomList():
	with open('roomList.pickle', 'rb') as f:
  		roomNumberList = pickle.load(f)
	return roomNumberList


#增量判断
def incrementRoom(roomNumberList):
	if(not os.path.isfile("roomList.pickle")):
		recordRoomList(roomNumberList)
		return roomNumberList
	existRoomList = readRoomList()
	for item in roomNumberList:
		if(item in existRoomList):
			continue
		else:
			existRoomList.append(item)
	recordRoomList(existRoomList)
	return existRoomList
```

### 2.2.2 获取主播id和竹子数

现在我们得到了房间号列表，根据我们抓包分析得到的地址，一个一个地请求。
```python
import requests

#得到该房间的主播id和竹子
def getBambooAndName(roomNumberList):
	roomInfoList = []
	prefix = "http://www.panda.tv/api_room_v2?roomid="
	for u in roomNumberList:
		singleRoom = []
		singleRoom.append(u)
		info = (requests.get(prefix + str(u))).json()
		if(info):
			print(info['data']['hostinfo']['name'])
			singleRoom.append(info['data']['hostinfo']['name'])
			singleRoom.append(round(int(info['data']['hostinfo']['bamboos'])/1e6, 2))
		else:
			singleRoom.append(["none", "none"])
			print("not get")
		roomInfoList.append(singleRoom)				
	return roomInfoList
```

请注意，此处我并没有简单地用两个list，而是使用了list in list的形式。why？这是因为为了后面的数据库而设计的。我这里的形式是[[room1],[room2],...]

## 2.3 存储到数据库Mongodb

首先需要安装mongodb数据库，然后我们语言是python，所以使用pymongo库。
[pymongo教程](http://api.mongodb.com/python/current/tutorial.html)

我们要以字典形式存储进数据库，而且要先设计好数据库，我的设计如下。
![mongodb框架设计](/images/mongodb_structure.jpg)

```python
from pymongo import MongoClient
from datetime import datetime

#获取字典形式
def getDict(roomInfoList):
	roomInfoDict = {}
	for item in roomInfoList:
		singleRoom = {"name": item[1], "bamboos": item[2], "date": datetime.utcnow()}
		roomInfoDict["room" + str(item[0])] = singleRoom
	return roomInfoDict

#存储到mongodb
def saveToMongo(roomInfoDict):
	client = MongoClient()
	db = client.pandaTV
	for k,v in roomInfoDict.items():
		collection = db[k]
		collection.insert(v)
	client.close()
```

最终存储进mongodb的数据是这样的，我们给每个post都加了一个：
```bash
> show collections
room10003
room10009
room100130
room10015
...

> db.room7000.find()
{ "_id" : ObjectId("57a2f91deccca320542a9e64"), "bamboos" : 21.4, "date" : ISODate("2016-08-04T08:13:16.973Z"), "name" : "LPL熊猫TV官方直播" }
{ "_id" : ObjectId("57a2ff93eccca31b38213ac1"), "name" : "LPL熊猫TV官方直播", "date" : ISODate("2016-08-04T08:40:51.872Z"), "bamboos" : 21.4 }
{ "_id" : ObjectId("57a45105eccca305c45cb33a"), "bamboos" : 21.64, "name" : "LPL熊猫TV官方直播", "date" : ISODate("2016-08-05T08:40:37.707Z") }
{ "_id" : ObjectId("57a5c283eccca30f6c94b0a0"), "bamboos" : 22.04, "name" : "LPL熊猫TV官方直播", "date" : ISODate("2016-08-06T10:57:07.491Z") }
...

```

这里要注意有个坑，需要先安装mongodb数据库，在运行爬虫之前，需要先开启数据库。
```bash
$ mongod
```

## 2.4 提取分析数据

关于这部分比较简单，大致就是读取数据库，按照时间戳来排序，得到最新的数据，然后返回Top20的一个Dict。特别需要注意的是，因为python的dict类型是无序的，所以即使你是排好序逐一添加，最后的Dict也是无序的。那么又因为我的Dict结构要方便数据可视化工具echarts.js读取，所以我设计成这样：
```python
Dict = {
	"name": ['xx', 'xx', ...]
	"bamboos": [xx, xx, ...]
}
```

我直接把有序的列表作为values，这样就可以不关注Dict的坑了。还有就是要两个列表组合排序的问题，我使用了zip和sorted。
```python
from pymongo import MongoClient
import json

def writeJsonFile(data,filename):
	with open(filename+".json", 'w') as f:
  		json.dump(data, f)

def get20Rank(db):
	bamboosList = []
	nameList = []
	rankDict = {}
	mycollections = db.collection_names()
	for item in mycollections:
		latest = db[item].find().sort('date',-1)[0]
		bamboosList.append(latest["bamboos"])
		nameList.append(latest["name"])
	bamboosList, nameList = zip(*sorted(zip(bamboosList, nameList), reverse=True))
	rankDict["name"] = nameList[0:19]
	rankDict["bamboos"] = bamboosList[0:19]
	writeJsonFile(rankDict, "bamboos_rank")
```

这样我们得到了包含数据的json文件。

## 2.5 数据可视化

### 2.5.1 echarts.js

关于数据可视化的选择有很多，web方面有highchart、echarts、d3之类。因为看中文爽，还是用了echarts。
[echarts官网](http://echarts.baidu.com/)
我们需要异步加载数据，[异步加载实例](http://echarts.baidu.com/tutorial.html#%E5%BC%82%E6%AD%A5%E6%95%B0%E6%8D%AE%E5%8A%A0%E8%BD%BD%E5%92%8C%E6%9B%B4%E6%96%B0)

这边还有个坑，就是因为x轴的数据太密集，会导致显示不全，解决办法：

```js
xAxis: [
    axisLabel: {
   	    interval: 0,
   	    rotate: 340
    },	
]
```

### 2.5.2 移动端简单支持

我使用了viewport来稍微支持了下移动端，即判断innerWidth的大小（显示屏）来设置参数。当屏幕宽度大于500px时，横向排版；小于500px时，纵向排版。

## 2.6 部署到网站

我们在调试echarts的时候，会遇到一个坑，即不能跨域，因为我们是读取本地json文件的。我安装了一个xampp来调试，比较方便。

因为我有一个配置好的虚拟主机，所以直接将json文件和html页面放上去就可以看到了。如果只是一个单纯的服务器，需要自己搭建服务器。或者放到github pages上也可以。

# 3. 总结

放上Github地址： [panda-bamboos-rank](https://github.com/HeroadZ/panda-bamboos-rank)  
展示地址： http://behappy.cc/extension/panda/rank.html

**PS: 由于熊猫直播已倒闭，本项目不再维护，展示地址不可访问。**

以上。

