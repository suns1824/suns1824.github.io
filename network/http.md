Http请求分为三部分：
>* Http请求行：Method  Request-URI HTTP-Version 
>* Http消息头（请求头）：Accept等等，Host，User-Agent，Content-length，Content-Type，Connection等
>* Http请求正文：可选的

1.请求方法
>* GET:获取Request-URI所标识的资源
>* POST:在Request-URI所标识的资源后附加新的提交数据
>* PUT：请求服务器存储一个资源，用RequestURI作为其标识
>* HEAD：请求获取由Request-URI所标识的资源的响应消息报头
>* DELETE：请求服务器删除RequestURI所标识的资源
>* 其他

2.post和get区别：
>* GET用于信息获取，而且是安全的和幂等的，POST则表示可能改变服务器上的资源的请求
>* GET请求的数据会附在URL之后，就是把数据放在请求行中，以？分隔URL和传输数据，多个参数用&连接；而POST提交会把提交的数据放置在HTTP消息的包体内，数据不会在地址栏中显示出来
>* 传输数据的大小不同。浏览器和服务器对URL长度有限制，所以XXX。
>* 安全性 GET提交数据，会出现在URL上，可能造成消息泄露；GET提交数据还会造成Cross-site request forgery攻击；POST提交的内容由于在消息体内传输，不存在上述安全问题

Http响应消息
>* 状态行 ： HTTP-Version Status-Code Reason-Phrase CRLF
>* 消息报头(响应报头)：允许服务器传递不能放在状态行中的附加响应信息XXX
>* 响应正文
  
[Http和Https](https://juejin.im/entry/58d7635e5c497d0057fae036)