### 1. Http的协议模型
```java
----------        ----------       -----------       ----------
| client | ---->  | 中间层1 | ----> | 中间层.. | ----> | server |
|        | <----  |        | <---- |         | <---- |        |
----------        ----------       -----------       ----------
    ^                ...                ...             ...                               
    |                  
    V                  
------------
| 可靠传输层 |
------------
```

概述：

* http是一个client/server的request-response标准，在client和server之间可能存在多个中间层
* http没有规定底层的协议，它只要求下层提供可靠的传输，事实上也就是TCP

#### non-persistent connection vs persistent connection
Http/1.1中新增了持久连接功能：服务器在发送响应后保持该TCP连接打开，相同的client-server之间的后续请求和响应报文能够通过相同的连接进行传送。

非持久化连接：每个TCP连接在服务器发送一个对象后关闭。

每一次连接最低消耗：至少2*RTT + 服务器端时间 +  client接受时间，很多情况下TCP的握手连接可能消耗了大部分时间
```java
+------+           +------+ ----
|Client| \  S      |Server|   ^
|      |   \ Y     |      |   |
|      |     \ N   |      |
|      |       >   |      |   RTT(Return Tip Time)
|      |       A  /|      |
|      |     C  /  |      |   |
|      |   K  /    |      |   V
|      |   <       |      | ----   
|      |\ SYN      |      |   ^
|      |  \ &&     |      |   |
|      |    \ Req  |      |   
|      |       >   |      |   RTT
|      |        R /|      |      
|      |      e /  |      |   |
|      |    s /    |      |   V
|      |   <       |      | ----
+------+           +------+

```
考虑一个Web页面包含一个基本的HTML页面和10个图片，总共需要11次请求：

* 如果使用非持久化连接，总共需要11个TCP连接，也就经历了11次上面的过程。注意：并不是说11个TCP连接是串行的，大部分浏览器默认支持5-10个并行TCP连接。
* 如果使用持久化连接，则11个对象可以使用1个TCP连接传输完。 

要足够重视TCP连接时间在整个请求响应时间的占比，可能会很大。参考《Web性能权威指南》第二章的论述。

###2. Http请求: request line, header, empty line, request data
```java
            +----------+----+-----+----+---------+----+----+
request line|  Method  | sp | URL | sp | Version | \r | \n |
            +----------+----+-----+----+---------+----+----+
            | header1: | sp |Value| \r |   \n    |
    header  +----------+----+-----+----+---------+ 
    fields  | header2: | sp |Value| \r |   \n    | 
            +----------+----+-----+----+---------+
            |      ......         |
            +----+----+-----------+
empty line  | \r | \n |
            +----+----+-----------------------------------+
            |              Request                        |
            |              Entity                         |
            +---------------------------------------------+ 

例子：
GET /a/214100687_355140 HTTP/1.1
Host: www.sohu.com
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Referer: https://www.baidu.com/link?url=c0qFsjJfTPB7nrOKfRwb4a4iGYM9nzDJ6cUcnFElAL1fIOrw1DsoROMCQR_Tea8N&wd=&eqid=c33660cf0001ec9f000000035ac2416d
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cookie: ad_t_3=1; ad_t_2=1; ad_t_side_fix_2=2; SUV=1710192016589445; vjuids=72a0d030f.15f907058a2.0.6538866c732f5; qch12=w:1; __utma=32066017.1335741029.1509956994.1509956994.1509956994.1; __utmz=32066017.1509956994.1.1.utmcsr=google|utmccn=(organic)|utmcmd=organic|utmctr=(not%20provided); sohutag=8HsmeSc5NCwmcyc5NCwmYjc5NSwmYSc5NywmZjc5NCwmZyc5NCwmbjc5NCwmaSc5NCwmdyc5NCwmaCc5NCwmYyc5NCwmZSc5NCwmbSc5NCwmdCc5NH0; IPLOC=CN1100; mut=zz.go.smuid; v=3; _smuid=WlF1aIQ8dmODk7bxJUTot; _smuid_type=2; gidinf=x099980107ee0d8c9e8892c4f000f0fc8f1e463a42e2; vjlast=1509956803.1521805643.12; t=1522680181117; reqtype=pc; beans_new_turn=%7B%22it-article%22%3A59%7D
（CXY: 空行）

--------
```

请求方法：

* GET 请求一个特定的资源，GET方法应仅仅是获取数据，而不能有副作用(server端可以做其他操作带来副作用，但是这些副作用不应有client负责)
* HEAD 同GET，但是返回的结果中没有response body。当获取资源的meta-information时比较有用，这样就不用传输所有的数据了
* POST 向指定URI提交数据，请求服务器进行处理
* PUT 向指定URI上传其最新内容
* DELETE 请求server删除Request-URI所标识的资源
* TRACE 回显服务器收到的请求，主要用于调试
* OPTIONS server传回该资源所支持的所有HTTP请求方法

### 3. Http Response
```java
            +----------+----+------------+----+---------------+----+----+
status line |  Version | sp | Status Code| sp | Reason Phrase | \r | \n |
            +----------+----+------------+----+---------------+----+----+
            | header1: | sp |   Value    | \r |      \n       |
    header  +----------+----+------------+----+---------------+ 
    fields  | header2: | sp |   Value    | \r |      \n       |
            +----------+----+------------+----+---------------+
            |                    ......                       |
            +----+----+---------------------------------------+
empty line  | \r | \n |
            +----+----+-----------------------------------+
            |              Response                       |
            |              Entity                         |
            +---------------------------------------------+ 

举例：
HTTP/1.1 200 OK
Content-Type: text/html;charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: nginx
Date: Mon, 02 Apr 2018 14:43:25 GMT
Cache-Control: max-age=300
X-From-Sohu: X-SRC-Cached
Content-Encoding: gzip
FSS-Cache: MISS from 6382512.11428794.7060733
FSS-Proxy: Powered by 2122607.2909049.2800763

（CXY： 返回实体为空）
```
返回状态码：

* 1xx消息: 请求已被服务器接收，继续处理
* 2xx成功: 请求已经被成功处理
* 3xx重定向: 需要后续操作才能完成这一请求，例外: 304表示资源没有更改
* 4xx请求错误
* 5xx服务器错误