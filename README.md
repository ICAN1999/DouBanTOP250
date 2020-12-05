# DouBanTOP250
豆瓣电影评分排名前250的电影信息
---
title: 爬取豆瓣电影TOP250
abbrlink: a7f672bc
date: 2020-09-25 16:33:51
tags:
  - 爬虫
categories:
  - [Python, 爬虫]
copyright: true
comments: true
---

看完小甲鱼的爬取豆瓣TOP250视频后，自己对代码进行了改写。

<!--more-->

## 准备工作

### 观察

要分自己观察爬取每个网页的信息，以免在后面的操作出现失误，比如：有的电影没有写上主演甚至主演就是一个省略号等一些情况（主要它的信息没有填写完整），这样更有利于我们编写代码。进入这个链接<https://movie.douban.com/top250>，然后可以发现每一页总共有 25 部电影，总共有十页，然后我们进入第二页时，发现链接变为<https://movie.douban.com?start=25&filter=>，进入到下一个页面时 start 的值就增加 25 直到 225，第一页 start 的值为 0。

### 尝试获取页面源代码

```
url = "https://movie.douban.com/top250"
r = requests.get(url)
r.encoding = r.apparent_encoding
print(r.status_code)
soup = BeautifulSoup(r.text, "html.parser")
print(soup.prettify())
```

结果却只得到 http 的状态码为 418，这种情况通常百度搜索，一般来说状态码为 200 时，才请求成功。肯定是服务器拒绝了爬虫的请求，一般来说有一下几个方法：

### 请求标头

设置 get 或者 request 方法（这个方法第一个参数是 method，表示 7 种请求的方式，第二个参数才是 url）里的 headers 参数。这个参数的类型为字典，键为"User-Agent"，值为当前网站提供你所用浏览器类型等信息的标识。<br>

```
url = "https://movie.douban.com/top250"
r = requests.get(url)
print(r.request.headers)
{'User-Agent': 'python-requests/2.24.0', 'Accept-Encoding': 'gzip, deflate', 'Accept': '*/*', 'Connection': 'keep-alive'}
```

这是没有设置这个参数的情况，网站通过"User-Agent"这个标识一下就认出这是爬虫程序并拒绝访问。当我们设置这个参数后情况就不一样了。

```
url = "https://movie.douban.com/top250"
hd = {'User-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.63'}
r = requests.get(url, headers=hd)
print(r.request.headers)
{'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.63', 'Accept-Encoding': 'gzip, deflate', 'Accept': '*/*', 'Connection': 'keep-alive'}
```

有了这个标识之后，网站向我们响应（也就是 response 对象）得到 request 请求里的 headers 就能看到"User-Agent",这样爬虫就可以隐藏自己的身份伪装成浏览器，让网站以为是人点击访问。

### 代理 ip

设置 proxies 使用代理服务器。使用爬虫爬取一个网站时，通常会频繁的访问该网站，网站会检测某一段时间某个 ip 的访问次数，如果访问次数过多，它会禁止你的访问。所有我们可以设置一些代理服务器来帮我们工作，每隔一段时间切换一个 ip，这样就不会出现禁止访问的现象。以上这两种方法我们可以使用 random 库的 choice 方法随机选择'User-Agent'、'http'或者'https'。

### cookie 认证

设置上述两个方法里的 cookies 参数，对应 Request 里的 cookie，这个可以检查网页对应链接的 Network 获得 Cookie。爬虫程序有了认证 cookie，服务器就会将它重定向到初始请求的资源，以免爬虫被引导到登录界面。如果多次之后还是失败的话，那就得考虑其他反爬虫机制的方法了。

### 如何得到 10 这个数字

找到页面下方 10 这个数字，鼠标左边点击检查，然后浏览器就会出现网站页面的源代码，并且还会自动跳转到 10 这个标签&lt;a href="?start=225&amp;filter="&gt;10&lt;/a&gt;，这 1~10 都是&lt;a&gt;超链接标签。

```
url = "https://movie.douban.com/top250"
hd = {'User-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.63'}
r = requests.get(url, headers=hd)
r.encoding = r.apparent_encoding
soup = BeautifulSoup(r.text, "html.parser")
depth = soup.find("a", href="?start=225&filter=")
print(depth.text)
```

结果为 10，但是类型为 str，这里要做一个强制类型转换。
还有另一种方法，比较麻烦又愚蠢而且容易懵，但我还是有个小细节想强调一下。可以看出&lt;a&gt;和&lt;span&gt;这两个标签都是&lt;div&gt;中，而它们又是同级标签，那么我们先找到&lt;a href="?start=225&amp;filter="&gt;10&lt;/a&gt;下的&lt;span&gt;标签，然后再平行遍历到&lt;a href="?start=225&amp;filter="&gt;10&lt;/a&gt;。

```
url = "https://movie.douban.com/top250"
hd = {'User-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.63'}
r = requests.get(url, headers=hd)
r.encoding = r.apparent_encoding
soup = BeautifulSoup(r.text, "html.parser")
depth = soup.find('span', class_='next').previous_sibling
print(depth)
```

结果却什么也没有，但是好像输出的空白比以往多了一行，原来是标签之间多了一个换行符，现在还真看不出来。

```
>>>depth = soup.find('span', class_='next')
>>>print(list(depth))
['\n', <link href="?start=25&amp;filter=" rel="next">
<a href="?start=25&amp;filter=">后页&gt;</a>
</link>]
```

那再加一个 previous_sibling 就可以得到 10，这里同样要类型转换。

```
depth = soup.find('span', class_='next').previous_sibling.previous_sibling
print(depth.text)
```

## 爬取网页

有了以上的准备我们就可以爬取网页的源代码了，废话不多说，直接上代码解释。

```
# 爬取网页代码
def get_html(url):
    # 使用代理
    iplist = ["121.232.148.225", "123.101.207.185", "69.195.157.162", "175.44.109.246", "175.42.128.246"]
    # random库的choice方法随机选择ip
    proxies = {"http": random.choice(iplist)}
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 \
        (KABUL, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.63',
        'cookie': 'bid=mDUxIx660Vg; douban-fav-remind=1; ll="118300"; '
                  '_vwo_uuid_v2=D0D095AA577CA9982D96BF53E5B82D902|19baada46789b48f67201d5ad830a0f6; '
                  '__yadk_uid=gAC153pUujwGBubBMhpUqg8osqjO7FfD; '
                  '__utmz=30149280.1601027331.15.8.utmcsr=accounts.douban.com|utmccn=('
                  'referral)|utmcmd=referral|utmcct=/passport/login; '
                  '__utmz=223695111.1601028074.12.5.utmcsr=accounts.douban.com|utmccn=('
                  'referral)|utmcmd=referral|utmcct=/passport/login; push_noty_num=0; push_doumail_num=0; ct=y; '
                  'ap_v=0,6.0; __utmc=30149280; __utmc=223695111; '
                  '__utma=30149280.846971982.1594690041.1601194295.1601199438.18; '
                  '__utma=223695111.2022617817.1595329698.1601194295.1601199438.15; __utmb=223695111.0.10.1601199438; '
                  '_pk_ses.100001.4cf6=*; _pk_ref.100001.4cf6=%5B%22%22%2C%22%22%2C1601199438%2C%22https%3A%2F'
                  '%2Faccounts.douban.com%2Fpassport%2Flogin%3Fredir%3Dhttps%253A%252F%252Fmovie.douban.com'
                  '%252Ftop250%22%5D; dbcl2="216449506:1anURmdR9nE"; ck=LxNQ; __utmt=1; __utmv=30149280.21644; '
                  'douban-profile-remind=1; '
                  '__gads=ID=c502f7a27a33facd-22f60957b9c300e3:T=1601201244:S=ALNI_MbN9wSYWbg2k5f3fWucwuXpORbGNw; '
                  '__utmb=30149280.20.10.1601199438; '
                  '_pk_id.100001.4cf6=d35078454df4f375.1595329083.15.1601201286.1601197192.'}
    res = requests.get(url, headers=headers, proxies=proxies, timeout=30)
    return res
```

定义一个获取网页源代码的函数，返回 Response 对象。这里有两种方法直接设置 cookie，一是添加到 headers，二是设置 cookies 参数。设置 timeout，timeout=([连接超时时间], [读取超时时间])，这里我们设置连接超时时间以免爬虫卡死又没有报错。

## 解析网页

解析网页是爬虫程序中最重要的一部分。我们已经得到了链接为<https://movie.douban.com/top250?start=0>下的网页源代码，然后我们就要解析 HTML 代码。使用 BeautifulSoup 库将它熬制成一碗美味汤，使用自带的 html.parser 解析器，对自定义的 get_html(url)函数返回的 Response 对象的 text 属性进行解析就可以得到源代码了。

```
soup = BeautifulSoup(res.text, 'html.parser')
```

剩下来的 9 页方法上基本差不多，注意一下个别信息缺失的电影，之后弄个循环就可以了。每部电影基本上有名称、评分、影评人数、导演、主演、上映时间、来自哪个国家和是什么类型的电影，还有对电影的描述。总共有这么多内容，接下就要用爬虫把它们一个个提取出来。定义一个函数，将提取出来的所有信息全部保存到一个列表，返回值为列表。

先上图，接下来我们要爬取信息大部分都在这个标签中。

![alt](/images/1.jpg)

### 电影名

```
movies = []
# 找到div标签
targets = soup.find_all("div", class_="hd")
for each in targets:
    try:
        # div标签下的a标签的下span标签的属性next才是电影名称，这
        三个标签是包含关系
        movies.append(each.a.span.text)
        # 为确保程序正常运行，这里添加一个异常处理。
    finally:
        continue
```

鼠标箭头放在电影名称上，然后左键就可以看到对应的标签，这个标签刚好是&lt;div class="hd"&gt;标签中的&lt;a href="https://movie.douban.com/subject/1292052/" class&gt;的第一个标签，因此我们先找到&lt;div class="hd"&gt;再找到&lt;span class="title">肖申克的救赎&lt;/span>这个标签，最后提取内容。使用 find_all 方法查找所有符合这个要求的标签并返回一个集合，然后将集合中的每个元素添加到列表中。

### 评分

```
ranks = []
targets = soup.find_all("span", class_="rating_num")
for each in targets:
    try:
        ranks.append(each.text)
    finally:
        continue
```

评分跟电影名都是类似这样的方法就可以得到。

### 导演

```
director = []
targets = soup.find_all("div", class_="bd")
for each in targets:
    try:
        # split方法分割字符串并返回分割后字符串的列表
        # lstrip去除左侧指定字符串
        director.append(each.p.text.split('\n')[1].strip().split('\xa0\xa0\xa0')[0].lstrip('导演: '))
    finally:
         continue
```

老方法找到导演的内容所在的标签，它在&lt;div class="bd"&gt;里的&lt;p class&gt;中，找到之后发现我们想要的信息都在这里面。接下来就要将&lt;p&gt;中的内容提取出来，内容为字符串类型，这里用 split 就整个字符串进行分割返回一个列表，然后选择正确的字符串并且用 strip 去掉无关的内容。

### 主演

```
leading_actor = []
targets = soup.find_all("div", class_="bd")
for each in targets:
    try:
        leading_actor.append(each.p.text.split('\n')[1].strip().split('\xa0\xa0\xa0')[1].split('主演:')[1])
    finally:
        continue
if movies[0] == "肖申克的救赎":
    leading_actor.insert(24, "无")
if movies[0] == "大闹天宫":
    leading_actor.insert(6, "无")
    leading_actor.insert(8, "无")
if movies[0] == "美国往事":
    leading_actor.insert(5, "无")
if movies[0] == "阳光姐妹淘":
    leading_actor.insert(23, "无")
if movies[0] == "七武士":
    leading_actor.insert(10, "无")
if movies[0] == "模仿游戏":
    leading_actor.insert(4, "无")
    leading_actor.insert(9, "无")
if movies[0] == "罗生门":
    leading_actor.insert(2, "无")
```

总共有 10 页，有些页的电影会缺少主演，如果这里没有 try、finally 异常处理的话，程序将会在这里报错：IndexError: list index out of range，意思索引超出范围，因为在 append()的这个括号里有不符合这个分割的字符串，分割之后并没有这个列表。

![我这里](/images/2.jpg)

就比如第一页最后一个电影《触不可及》，不能以"主演:"分割，返回不了一个列表也就没有 split\("主演:"\)\[1\]这个索引了。之后在每次循环找到对应的缺失页后就往列表里插入缺失的信息，保证 10 次循环的每次 leading_actor 的长度都为 25，因为每页总共 25 电影。

### 评价人数

```
people = []
targets = soup.find_all("div", class_="star")
for each in targets:
    try:
        people.append(str(each.contents[7].text).rstrip("人评价"))
    finally:
        continue
```

将&lt;div&gt;标签下的的所有儿子节点（contents）索引为 7 的 text 转化为 str 并去掉"人评价"，最后存入列表。

### 电影描述

```
# 电影描述，最后两页分别有两篇电影没有描述
sentence = []
targets = soup.find_all("span", class_="inq")
for each in targets:
    try:
        sentence.append(str(each.text))
    finally:
        continue
if sentence[22] == "天使保护事件始末。":
    sentence.insert(9, "无")
    sentence.insert(14, "无")
if sentence[22] == "生病的E.T.皮肤的颜色就像柿子饼。":
    sentence.insert(6, "无")
    sentence.insert(22, "无")
```

以缺失页的最后一个电影描述为标记，在列表中插入“无”，代表该电影没有写上描述，保证每次循环 sentence 的长度为 25。

### 上映时间

```
year = []
targets = soup.find_all("div", class_="bd")
for each in targets:
     try:
        year.append(each.p.text.split('\n')[2].strip().split('\xa0')[0].strip("(中国大陆)"))
    finally:
        continue
    if movies[0] == "大闹天宫":
        year[0] = "1961 / 1964 / 1978 / 2004"
```

利用这个异常处理去掉《天书奇谭》年份里的字符串"中国大陆"。还有一部电影《大闹天宫》的上映年份分别为 1961、1964、1978、2004 这四年，只需修改该网页下第一个 year[0]的值就行。

### 国家

```
country = []
targets = soup.find_all("div", class_="bd")
for each in targets:
    try:
        country.append(each.p.text.split('\n')[2].strip().split('\xa0')[2])
    finally:
        continue
```

### 类型

```
kinds = []
targets = soup.find_all("div", class_="bd")
for each in targets:
    try:
        kinds.append(each.p.text.split('\n')[2].strip().split('\xa0')[4])
    finally:
        continue
```

这两个跟之前的是同一个道理。

### 整合数据

```
result = []
length = len(movies)
for i in range(length):
    result.append([movies[i], ranks[i], director[i], leading_actor[i],people[i], sentence[i], year[i], country[i], kinds[i]])
return result
```

length 的值为 25，有的电影信息缺失了，也是这些储存电影信息的列表不为 25，在这里将会报 IndexError，所以得确保这些储存信息的列表每次循环之后的长度都为 25，利用循环将每部电影的信息全都保存到 result 里，并作为函数返回值返回。

## 保存数据

```
def save_to_excel(result):
    # 实例化一个对象，相当于创建一个Excel文档
    wb = Workbook()
    # 激活Sheet
    ws = wb.active
    # 设置标题
    ws["A1"] = "名称"
    ws["B1"] = "评分"
    ws["c1"] = "导演"
    ws['d1'] = "主演"
    ws["e1"] = "评价人数"
    ws["f1"] = "电影描述"
    ws["g1"] = "年份"
    ws["h1"] = "国家"
    ws["i1"] = "类型"
    # 信息添加ws中
    for each in result:
        ws.append(each)
    # 保存文件
    wb.save("豆瓣电影TOP250.xlsx")
```

这里用到 python 的第三方模块 openpyxl，通过 pip 命令安装。

## main

```
def main():
    host = "https://movie.douban.com/top250"
    res = get_html(host)
    depth = find_pages(res)
    result = []
    for i in range(depth):
        url = host + '/?start=' + str(25 * i)
        res = get_html(url)
        result.extend(parser_html(res))
    save_to_excel(result)
```

利用 find_pages(res)获得网页页数，循环 depth 次，每次访问一个链接并获取内容，然后添加到 result 中，最后保存为 excel 格式的文件。

## 程序代码

```
import requests
from bs4 import BeautifulSoup
import random
from openpyxl import Workbook


def get_html(url):
    # 使用代理
    iplist = ["121.232.148.225", "123.101.207.185", "69.195.157.162", "175.44.109.246", "175.42.128.246"]
    proxies = {"http": random.choice(iplist)}
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 \
        (KABUL, like Gecko) Chrome/85.0.4183.121 Safari/537.36 Edg/85.0.564.63',
        'cookie': 'bid=mDUxIx660Vg; douban-fav-remind=1; ll="118300"; '
                  '_vwo_uuid_v2=D0D095AA577CA9982D96BF53E5B82D902|19baada46789b48f67201d5ad830a0f6; '
                  '__yadk_uid=gAC153pUujwGBubBMhpUqg8osqjO7FfD; '
                  '__utmz=30149280.1601027331.15.8.utmcsr=accounts.douban.com|utmccn=('
                  'referral)|utmcmd=referral|utmcct=/passport/login; '
                  '__utmz=223695111.1601028074.12.5.utmcsr=accounts.douban.com|utmccn=('
                  'referral)|utmcmd=referral|utmcct=/passport/login; push_noty_num=0; push_doumail_num=0; ct=y; '
                  'ap_v=0,6.0; __utmc=30149280; __utmc=223695111; '
                  '__utma=30149280.846971982.1594690041.1601194295.1601199438.18; '
                  '__utma=223695111.2022617817.1595329698.1601194295.1601199438.15; __utmb=223695111.0.10.1601199438; '
                  '_pk_ses.100001.4cf6=*; _pk_ref.100001.4cf6=%5B%22%22%2C%22%22%2C1601199438%2C%22https%3A%2F'
                  '%2Faccounts.douban.com%2Fpassport%2Flogin%3Fredir%3Dhttps%253A%252F%252Fmovie.douban.com'
                  '%252Ftop250%22%5D; dbcl2="216449506:1anURmdR9nE"; ck=LxNQ; __utmt=1; __utmv=30149280.21644; '
                  'douban-profile-remind=1; '
                  '__gads=ID=c502f7a27a33facd-22f60957b9c300e3:T=1601201244:S=ALNI_MbN9wSYWbg2k5f3fWucwuXpORbGNw; '
                  '__utmb=30149280.20.10.1601199438; '
                  '_pk_id.100001.4cf6=d35078454df4f375.1595329083.15.1601201286.1601197192.'}
    res = requests.get(url, headers=headers, proxies=proxies, timeout=30)
    # res = requests.get(url, headers=headers)
    return res


def parser_html(res):
    soup = BeautifulSoup(res.text, 'html.parser')

    # 电影名
    movies = []
    targets = soup.find_all("div", class_="hd")
    for each in targets:
        try:
            movies.append(each.a.span.text)
        finally:
            continue

    # 评分
    ranks = []
    targets = soup.find_all("span", class_="rating_num")
    for each in targets:
        try:
            ranks.append(each.text)
        finally:
            continue

    # 导演
    director = []
    targets = soup.find_all("div", class_="bd")
    for each in targets:
        try:
            director.append(each.p.text.split('\n')[1].strip().split('\xa0\xa0\xa0')[0].lstrip('导演: '))
        finally:
            continue

    # 主演
    leading_actor = []
    targets = soup.find_all("div", class_="bd")
    for each in targets:
        try:
            leading_actor.append(each.p.text.split('\n')[1].strip().split('\xa0\xa0\xa0')[1].split('主演:')[1])
        finally:
            continue
    if movies[0] == "肖申克的救赎":
        leading_actor.insert(24, "无")
    if movies[0] == "大闹天宫":
        leading_actor.insert(6, "无")
        leading_actor.insert(8, "无")
    if movies[0] == "美国往事":
        leading_actor.insert(5, "无")
    if movies[0] == "阳光姐妹淘":
        leading_actor.insert(23, "无")
    if movies[0] == "七武士":
        leading_actor.insert(10, "无")
    if movies[0] == "模仿游戏":
        leading_actor.insert(4, "无")
        leading_actor.insert(9, "无")
    if movies[0] == "罗生门":
        leading_actor.insert(2, "无")

    # 评价人数
    people = []
    targets = soup.find_all("div", class_="star")
    for each in targets:
        try:
            people.append(str(each.contents[7].text).rstrip("人评价"))
        finally:
            continue

    # 电影描述，最后两页分别有两篇电影没有描述
    sentence = []
    targets = soup.find_all("span", class_="inq")
    for each in targets:
        try:
            sentence.append(str(each.text))
        finally:
            continue
    if sentence[22] == "天使保护事件始末。":
        sentence.insert(9, "无")
        sentence.insert(14, "无")
    if sentence[22] == "生病的E.T.皮肤的颜色就像柿子饼。":
        sentence.insert(6, "无")
        sentence.insert(22, "无")

    # 年份
    year = []
    targets = soup.find_all("div", class_="bd")
    for each in targets:
        try:
            year.append(each.p.text.split('\n')[2].strip().split('\xa0')[0].strip("(中国大陆)"))
        finally:
            continue
    if movies[0] == "大闹天宫":
        year[0] = "1961 / 1964 / 1978 / 2004"
    # 国家
    country = []
    targets = soup.find_all("div", class_="bd")
    for each in targets:
        try:
            country.append(each.p.text.split('\n')[2].strip().split('\xa0')[2])
        finally:
            continue
    # 类型
    kinds = []
    targets = soup.find_all("div", class_="bd")
    for each in targets:
        try:
            kinds.append(each.p.text.split('\n')[2].strip().split('\xa0')[4])
        finally:
            continue
    result = []
    length = len(movies)
    for i in range(length):
        result.append([movies[i], ranks[i], director[i], leading_actor[i],
                       people[i], sentence[i], year[i], country[i], kinds[i]])
    return result


# 找出一共有多少个页面
def find_pages(res):
    soup = BeautifulSoup(res.text, 'html.parser')
    depth = soup.find('span', class_='next').previous_sibling.previous_sibling.text

    return int(depth)


def save_to_excel(result):
    wb = Workbook()
    ws = wb.active
    ws["A1"] = "名称"
    ws["B1"] = "评分"
    ws["c1"] = "导演"
    ws['d1'] = "主演"
    ws["e1"] = "评价人数"
    ws["f1"] = "电影描述"
    ws["g1"] = "年份"
    ws["h1"] = "国家"
    ws["i1"] = "类型"
    for each in result:
        ws.append(each)
    wb.save("豆瓣电影TOP250.xlsx")


def main():
    host = "https://movie.douban.com/top250"
    res = get_html(host)
    depth = find_pages(res)
    result = []
    for i in range(depth):
        url = host + '/?start=' + str(25 * i)
        res = get_html(url)
        result.extend(parser_html(res))
    save_to_excel(result)


if __name__ == "__main__":
    main()
```

## 部分结果展示

![我在这里](/images/3.jpg)
