
# [HTTP](https://juejin.cn/post/6844904100035821575#heading-17)

## 报文结构
- 请求行（Get...URI...Version）+请求头（Key Value）+请求体（Body 数据）

## HTTP 特点缺点
- 扩展强，只是简单规定了基本结构；
- 可靠传输：基于 TCP 可以记录那些数据被传输；


- 无状态：
- 明文传输

## Accept 系列字段
- 数据格式 MIME：发送方指定 Content-Type；接收方 Accept-Type 字段标明接收类型；
- 压缩方式：Content-Encoding 发送方，Accept-Encoding 接收方；
- 支持语言：Content-Language 发送发 对应；
- 字符集：Content-Charset:utf-8

## 大文件传输
- **范围请求**：Accept-Ranges: none 表示无限接收范围

## 表单
- 是一种 Content-Type `key=1&key2=2`

# [GET和POST的区别](https://blog.csdn.net/guorui_java/article/details/112294323)

## 直观区别
- Get 请求数据放在 URL 里面、Post 放在 Body；
- Get 数据长度有限制，2K 服务器限制 64k，但也不是强制标准；
- Get 操作是幂等的；
- Get 只处理 ASCII 字符，Post 不限制；
  
## 本质区别
- 本质没有区别；
- 都是 HTTP 协议的传输，Get 也可以拥有 Body 只要服务器去读取、Post 也可以塞数据到 url；
- 之所以产生区别，是人为协议，不同的传输贴上标签 方便解析；

## 重大区别
- TCP 数据包：Get 只产生一个数据包，直接由客户端到服务端。Post 两个：Header 发一个，服务器响应 100 continue 继续发送 data，服务器返回 200；


