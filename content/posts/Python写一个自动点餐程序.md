---
categories: ["技术", "Python"]
title: Python写一个自动点餐程序
date: 2020-06-02 10:17:12
tags: ["前后端"]
---
# Python写一个自动点餐程序

## 为什么要写这个

公司现在用meican作为点餐渠道，每天规定的时间是早7：00-9：40点餐，有时候我经常容易忘记，或者是在地铁/公交上没办法点餐，所以总是没饭吃，只有去楼下711买点饭团之类的玩意儿，所以这是促使我写点餐小程序的原因。

## 点餐的流程

登录 ---> 点餐 ---> 提交  

哈哈，是不是很简单，其实这个还好，说白了，就是登录上去，然后拿到cookie，保持一个登录状态，然后再去点餐，点餐就是构造请求，发送到指定的点餐URL上就可以了。

## 登录

首先我们点开  

 https://meican.com/  
![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/python%E7%82%B9%E9%A4%90/1082248-20190809104417503-905884945.png)

上面要求我们登录，我们这里输入自己的账号密码，登录上去之后可以看见一个请求. 
![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/python%E7%82%B9%E9%A4%90/1082248-20190809104846744-1053796310.png)


这个请求就是登录的请求，我们看下需要传什么参数，然后我们去完全构造这个请求，也就是参数一致，并且带浏览器头,这里我们也需要去保存cookie，也就是说，我们需要自己的账号时刻保持online状态，所以需要保存cookie，需要时候调用  

所以我们需要实现如下功能

1. 登录请求构造
   
2. 保持登录状态
   
3. 保存cookies
   
4. 使得后来的访问都带cookie  

代码如下

```python
import json
import requests
import http.cookiejar as HC
session = requests.session()
session.cookies = HC.LWPCookieJar(filename='cookies')
def login_meican():
    """
    登录美餐，寻找cookie文件，没cookie文件就重新载入
    :return:
    """
    # 储存cookie作为日后使用，三天clear一次
    try:
        session.cookies.load(ignore_discard=True)
    except:
        print('未找到cookies文件')
        save_cookie()

def save_cookie():
    """
    如果没cookie，登录逻辑
    :return:
    """
    login_url = 'https://meican.com/account/directlogin'

    # Headers
    hearsers = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36",
        "Referer": "https://meican.com/login",
        "Origin": "https://meican.com",
        "Host": "meican.com",
        "Accept": "*/*"
    }

    # Login need data

    data = {
        "username": "xxxxxxxxxxx",
        "loginType": "username",
        "password": "xxxxxxxxxxx",
        "remember": "true"
    }

    try:
        r = session.post(login_url, headers=hearsers, data=data)
        r.raise_for_status()
        session.cookies.save()
    except Exception as e:
        print("login error!")
        return 0
```

上面的代码实现了登录。

## 点餐

### 找到菜单

这里需要找到菜单，因为截图忘了截，这里就直接公布吧，找到菜单需要两个参数，一个是uuid，另一个是addrid，也就是你登陆的凭证+你所在地区的id，没有这两个是无法找出菜单的，并且也无法继续点餐流程。  

#### 如何获得这两个参数

在登录的时候我发现了一个URL，这个URL是 **https://meican.com/preorder/api/v2.1/calendaritems/list?withOrderDetail=false&beginDate=2019-09-04&endDate=2019-09-04**，  

这个URL下的返回有我们要的参数，uuid 和 addrid，所以构造请求去获取这两个参数

```python
def get_for_my_order():
    """
    找到usertorken, addrid
    :return:
    """
    user_dict = {}
    Now_date = datetime.date.today()
    z = session.get("https://meican.com/preorder/api/v2.1/calendaritems/list?withOrderDetail=false&beginDate={Now}&endDate={Now}".format(Now=Now_date))
    x = json.loads(z.text)
    user_dict["uuid"] = x["dateList"][0]["calendarItemList"][0]["userTab"]["uniqueId"]
    user_dict["addrid"] = x["dateList"][0]["calendarItemList"][0]["userTab"]["corp"]["addressList"][0]["uniqueId"]
    return user_dict
```
#### 构造获取菜单请求

找到获取菜单的URL  

 https://meican.com/preorder/api/v2.1/recommendations/dishes?tabUniqueId={uuid}&targetTime={Now}+09:40  

这里需要一个参数uuid，调取我们获取参数的函数

```python
def get_menu():
    """
    获取餐单逻辑
    :return:
    """
    menu_dict = {}
    menu_list = []
    Now_date = datetime.date.today()
    uuid = get_for_my_order()["uuid"]
    z = session.get("https://meican.com/preorder/api/v2.1/recommendations/dishes?tabUniqueId={uuid}&targetTime={Now}+09:40".format(uuid = uuid, Now=Now_date))
    menu = json.loads(z.text)["myRegularDishList"]
    for i in menu:
        menu_dict["id"] = i["id"]
        menu_dict["name"] = i["name"]
        z = copy.deepcopy(menu_dict)
        menu_list.append(z)
    return menu_list
```

输出所有的菜单，以一个list作为输出

## 提交

### 构造点餐请求

首先先找到点餐的URL  

 https://meican.com/preorder/api/v2.1/orders/add  

 查看点餐需要的参数：

 ```python
     data = {
        "corpAddressUniqueId": addrid,
        "order": x,
        "remarks": y,
        "tabUniqueId": uuid,
        "targetTime":target_time,
        "userAddressUniqueId":addrid
    }
```

构造点餐请求

```python
def order_action():
    """
    点餐逻辑
    :return:
    """
    addrid = get_for_my_order()["addrid"]
    uuid = get_for_my_order()["uuid"]
    menu_list = get_menu()
    menu_id = choice(menu_list)["id"]
    target_time = str(datetime.date.today()) + " " + "09:40"
    x = str([{"count":1,"dishId":menu_id}])
    y = str([{"dishId":menu_id,"remark":""}])

    data = {
        "corpAddressUniqueId": addrid,
        "order": x,
        "remarks": y,
        "tabUniqueId": uuid,
        "targetTime":target_time,
        "userAddressUniqueId":addrid
    }

    headers = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36"
    }
    try:
        z = session.post("https://meican.com/preorder/api/v2.1/orders/add", headers=headers, data=data)
        z.raise_for_status()
    except:
        return "点餐错误！"
```

## 所用的知识点一览

1. Python requetst的post，session
   
2. cookie的保存和调用
   
3. json的输出和浏览
   
4. random.choice 的列表元素随机选择
   
5. Python构造请求和登录逻辑