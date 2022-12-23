# 关于本项目
原项目地址：https://github.com/boy-hack/hack-requests<br/>
作者貌似不维护了，所以自己fork并进行了修改，修复了一个BUG，一个更改，添加了两个功能<br/>
有问题可以提交issues或邮箱kasterborous@qq.com找我

## BUG
在调用.close()关闭连接时，不知道出现了什么错误，会导致响应包内容丢失，我在调用.close()之前先将响应包内容复制并保存，以防止内容丢失，后续使用这个复制出来的内容即可

## 更改
<strong>旧：</strong>httpraw中的Connection固定值为close<br/>
<strong>新：</strong>Connection的值默认为close，如果用户自定义了Connection，则使用自定义的值

## 新功能1
<strong>旧：</strong>单个http请求不支持自定义timeout超时，固定为17秒<br/>
<strong>新：</strong>单个http请求支持自定义timeout超时，如果指定timeout则使用指定的值，如果不指定的话默认为17秒

```python
import HackRequests

hack = HackRequests.hackRequests(timeout=10)

res = hack.http('https://www.baidu.com')
print(res.status_code)
```

以上代码输出结果：200

```python
import HackRequests

hack = HackRequests.hackRequests(timeout=0.01)

res = hack.http('https://www.baidu.com')
print(res.status_code)
```

以上代码报错，socket连接超时：socket connect timeout

## 新功能2
<strong>旧：</strong>线程池threadpool支持http和httpraw，只能添加url或raw进行单一请求，不能进行判断等操作<br/>
<strong>新：</strong>线程池threadpool支持http、httpraw和httpfunc，可以添加url、raw或自定义函数，你可以将POC的流程写入函数（循环、请求、判断、return等），然后通过httpfunc将POC函数添加到线程池当中，最后通过callback回调函数获取return的值

```python
import HackRequests

results = []

def _callback(result):
    results.append(result)

def func(url):
    return url

threadpool = HackRequests.threadpool(10, callback=_callback, timeout=10)
for i in range(5):
    threadpool.httpfunc(func, 'https://www.baidu.com')

threadpool.run()

print(results)
```

以上代码
* 首先定义一个列表results，用于存储return的结果
* 定义一个回调函数_callback，函数名称可以自定义，最少接收一个参数，该参数是线程池中 单个函数的返回值，在函数中 将单个返回值添加到results中
* 定义一个函数func，函数名称可以自定义，这个函数是你实际定义的函数，最少接收一个参数，该参数是url，在函数中将url通过return返回
* 新建一个线程池threadpool，第一个参数是线程，第二个参数是回调函数，第三个参数是超时时间/秒
* 通过for循环和httpfunc方法，往线程池里面添加5个函数，第一个参数是函数名称（注意没有括号），第二个参数是url
* 通过run()方法运行线程池
* 最后打印results中的内容，输出结果为：
['https://www.baidu.com',
'https://www.baidu.com',
'https://www.baidu.com',
'https://www.baidu.com',
'https://www.baidu.com']

```python
# 一个简单的请求演示
import HackRequests

results = []
timeout = 10

def _callback(result):
    results.append(result)

def POC(url, timeout):
    hack = HackRequests.hackRequests(timeout=timeout)
    res = hack.http(url)
    
    return res.status_code

threadpool = HackRequests.threadpool(10, callback=_callback, timeout=timeout)
for i in range(5):
    threadpool.httpfunc(POC, 'https://www.baidu.com', timeout=timeout)

threadpool.run()

print(results)
```

以上代码输出结果：[200, 200, 200, 200, 200]

# hack-requests
HackRequests 是基于`Python3.x`的一个给黑客们使用的http底层网络库。如果你需要一个不那么臃肿而且像requests一样优雅的设计，并且提供底层请求包/返回包原文来方便你进行下一步分析，如果你使用Burp Suite，可以将原始报文直接复制重放，对于大量的HTTP请求，hack-requests线程池也能帮你实现最快速的响应。

- 像requests一样好用的设计
- 提供接口获得底层请求包、返回包原文，方便下一步分析
- 支持发送HTTP原始报文，支持从Burp Suite等抓包软件中重放
- hack-requests是单文件模块，可方便移植到其他项目中。

## 安装
- 仅支持python3
- pip install HackRequests

## 特征

### 不需要关注参数类型

在`requests`模块中，为了方便使用，header、cookie、post等信息都是以字典形式传参，但对于黑客来说，常常截获到的是一个文本，手动转换成字典费时费力。但在`HackRequests`中，这些参数你既可以传入一个字典，也可以传入一个文本，程序会自动识别并转换。

```python
import HackRequests

hack = HackRequests.hackRequests()
url = "http://x.hacking8.com"

header = '''
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
'''
hh = hack.http(url, headers=header)
print(hh.text())

headers = {
    "Connection": "keep-alive",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36"
}
uu = hack.http(url, headers=headers)
print(uu.text())

```

### 提供底层包分析

`hackRequests`返回结果中带有`log`参数，记录了`request`和`response`，在写扫描器的时候这两个参数参考非常重要。

```python
import HackRequests

hack = HackRequests.hackRequests()
url = "http://x.hacking8.com"

header = '''
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
'''
hh = hack.http(url, headers=header)
print(hh.log.get("request"))
print()
print(hh.log.get("response"))

```

返回

```bash
GET / HTTP/1.1
Host: x.hacking8.com
Connection: Keep-Alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8

HTTP/1.1 200 OK
Server: nginx
Date: Sat, 01 Sep 2018 12:52:35 GMT
Content-Type: text/html
Content-Length: 580
Last-Modified: Thu, 16 Aug 2018 09:50:56 GMT
Connection: keep-alive
ETag: "5b754900-244"
Accept-Ranges: bytes
```

### BurpSuite 重放

支持直接将代理抓包软件中的请求

```python
import HackRequests

hack = HackRequests.hackRequests()
raw = '''
GET / HTTP/1.1
Host: x.hacking8.com
Connection: Keep-Alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
'''
hh = hack.httpraw(raw)
print(hh.text())
```

### 内置线程池

在并发网络访问方面，HackRequests的线程池可以帮助您把网络并发优化到极致。 
```python
import HackRequests


def _callback(r:HackRequests.response):
    # 从回调函数取出结果，参数r是response结果
    print(r.text())


threadpool = HackRequests.threadpool(threadnum=10,callback=_callback)
url = "http://www.baidu.com"
for i in range(50):
    threadpool.http(url)
threadpool.run()
```

### 学习方面

代码开源，不到500行代码且中文注释可以帮助你更好的理解该项目思路。

## 说明文档

### 快速使用

```python
import HackRequests
hack = HackRequests.hackRequests()
url = "http://www.baidu.com/index.php"
u = hack.http(url,method="HEAD")
```

说明：`HEAD`模式可以帮助你更快的检测网页是否存在

使用`hack.http()`可以填写下列参数，当然，除了`url`参数外都不是必须的。

| 参数名      | 参数功能                                                     | 参数类型 |
| ----------- | ------------------------------------------------------------ | -------- |
| url（必须） | 用于传递一个地址                                             | Str      |
| post        | post参数用于传递post提交，此参数被选择时，`method`自动变为`POST`,post参数的类型可以为`Str`或者`Dict` | Str/Dict |
| method      | 访问模式，目前支持三种 HEAD、GET、POST，默认为GET            | Str      |
| location    | 当状态码为301、302时会自动跳转，默认为True                   | Bool     |
| proxy       | 代理，需要传入一个tuple，类似 ('127.0.0.1','8080')           | Tuple    |
| headers     | 自定义HTTP头，可传入字典或原始的请求头                       | Str/Dict |
| cookie      | 自定义Cookie，可传入字典或原始cookie字符串                   | Str/Dict |
| referer     | 模拟用户Referer                                              | Str      |
| user_agent  | 用户请求头，若为空则会模拟一个正常的请求头                   | Str      |
| real_host   | 用于host头注入中在header host字段填写注入语句，这里填写真实地址 如 "127.0.0.1:8000"  具体参考：https://github.com/boy-hack/hack-requests/blob/master/demo/CVE-2016-10033.py | str      |

### 发送原始响应头

使用`hackRequests`中的 `httpraw`方法

```python
import HackRequests

hack = HackRequests.hackRequests()
raw = '''
GET / HTTP/1.1
Host: x.hacking8.com
Connection: Keep-Alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
'''
hh = hack.httpraw(raw)
print(hh.text())
```

| 参数名    | 参数类型 | 参数功能                                                     |
| --------- | -------- | ------------------------------------------------------------ |
| raw(必须) | Str      | 原始报文                                                     |
| ssl       | Bool     | 网站是否是https，默认为False                                 |
| proxy     | Tuple    | 代理地址                                                     |
| location  | Bool     | 自动跳转，默认为Ture                                         |
| real_host | str      | 用于host头注入中在header host字段填写注入语句，这里填写真实地址 如 "127.0.0.1:8000"  具体参考：https://github.com/boy-hack/hack-requests/blob/master/demo/CVE-2016-10033.py |

注:httpraw方法最后会解析格式到`http`方法,所以`http`方法使用的参数这里都可以使用

### response

可使用如下接口获取`hack.http()`的返回值

| 接口参数    | 功能                            | 返回值类型 |
| ----------- | ------------------------------- | ---------- |
| status_code | 获取返回状态码                  | Int        |
| content()   | 获取返回字节                    | Bytes      |
| text()      | 获取返回文本(会自动转码)        | Str        |
| header      | 返回原始响应头                  | Str        |
| headers     | 返回原始响应头的字典形式        | Dict       |
| charset     | 获取编码类型                    | Str        |
| log         | 获取底层发送的请求包/返回包     | Dict       |
| url         | 返回url，若发生跳转则为跳转后的 | Str        |
| cookie      | 返回请求后的Cookie           | Str        |
| cookies     | 返回请求后的Cookie字典形式    | Dict       |

### 线程池

```python
import HackRequests


def _callback(r:HackRequests.response):
    # 从回调函数取出结果，参数r是response结果
    print(r.text())


threadpool = HackRequests.threadpool(threadnum=10,callback=_callback,timeout=10)
# 可设置http访问的超时时间，不设置则默认为10s。线程数量[threadnum]设置根据自己电脑配置设置，默认为10,值越大线程越多同一秒访问的网站数量也越多。
url = "http://www.baidu.com"
for i in range(50):
    threadpool.http(url)
threadpool.run()
```
回调函数参数r是response类，见[说明文档]-[response]  
在声明一个线程池为`threadpool`后，有以下三种方法可以调用

| 方法名    | 传入参数                | 功能                               |
| --------- | ----------------------- | ---------------------------------- |
| http()    | 见[说明文档]-[快速使用] | 将HTTP请求后加入现成队列，准备执行 |
| httpraw() | 见[说明文档]-[快速使用] | 将HTTP请求后加入现成队列，准备执行 |
| stop()    |                         | 停止线程池                         |
| run()     |                         | 启动线程池                         |


