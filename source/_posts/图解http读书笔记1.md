---
layout: post
title: 图解http读书笔记1 
tags: [读书笔记, http]
category: read
---

这是自己读*图解http*的读书笔记，没有读 *TCP/IP详解 卷1*是因为考虑到自己的时间，想碰到具体深入底层的问题时再去查阅。这篇包含了http报文首部和http状态码，这两部分其实放到一起更易贯通。有一些状态码是针对特定首部字段才会出现的。

<!-- more -->

### HTTP报文首部

http报文首部分为请求报文和响应报文。请求报文包含请求行，请求首部字段，通用首部字段，实体首部字段；响应报文包含状态行，响应首部字段，通用首部字段，实体首部字段。其中请求行包含方法（get，post等）、URI、HTTP版本，状态行包含HTTP版本、状态码。各个首部字段将具体介绍


#### 通用首部字段

1. Cache-Control 进行缓存控制。no-cache是为了防止从缓存中返回过期的资源，客户端的请求中包含它表示客户端不会接收缓存过的响应，服务器返回的响应中包含no-cache，缓存服务器不能对资源进行缓存；no-store是不进行缓存；然后就是一些具体的指定缓存期限和认证的指令如max-age、min-fresh等
2. Connection 主要用作控制不再转发给代理的首部字段和管理持久连接。
	
	HTTP首部将定义成缓存代理和非缓存代理的行为，分成了2种。端到端首部（end-to-end）：此类别的首部会发送到最终的接收目标，且必须保存在缓存生成的响应中，必须被转发。逐跳首部(hop-by-hop)：只对单次转发有效，会因为缓存和代理而不再转发。如果要使用hop-by-hop首部，需要提供Connection首部。Connection,Keep-A live,Proxy -A uthenticate,Proxy -A uthorization,Trailer,TE,Transfer-Encoding,Upgrade这八个是hop-by-hop。
	
	管理持久连接上面主要由Keep-Alive和Close。http／1.1默认的是持久连接，http／1.1之前的默认是非持久连接
	
3. Date 创建http报文的日期和时间
4. Pragma 是http／1.1之前版本的遗留字段，仅作为向后兼容而定义。使用形式如下：Pragma：no-cache。只用在客户端发送的请求中。在http／1.1中直接采用Cache-Control：no-cache，兼容以前版本一般也会加上Pragma：no-cache
5. Trailer 事先说明了在报文主体后记录了哪些首部字段。该首部字段可应用在http／1.1版本分块传输编码时。
6. Transfer—Encoding 规定了传输报文主体时采用的编码方式，仅对分块传输编码有效。
7. Upgrade 用于检测http协议及其他协议是否可使用更高的版本进行通信。Upgrade首部字段产生作用的Upgrade对象仅限于客户端和临接服务器。
8. Via 追踪客户端与服务器之间的请求和响应报文的传输路径。可以避免请求回环的发生，在经过代理时必须附加该首部字段内容。各个代理服务器会往via首部添加自身的服务器信息。
9. Warning 从http／1.0的Retry-After演变过来的。通常会告知用户一些与缓存相关的问题警告。

#### 请求首部字段

1. Accept 客户端能够处理的媒体类型及媒体类型的相对优先级。优先级权重范围0-1，默认1.0
2. Accept-Charset 客户端支持的字符集及字符集的相对优先级。字符集指：iso-8859-5，unicode等
3. Accept-Encoding 客户端支持的内容编码及内容编码的优先级。内容编码指：gzip，compress，deflate等
4. Accept-Language 支持语言及语言优先级。
5. Authorization 认证信息，通常会在收到服务器的401之后，把Authorization及认证信息加入请求中。
6. Expect 告知服务器，期望出现某种特定行为。服务器无法满足期望，会返回417
7. From 使用用户代理的用户的电子邮件地址。
8. Host http／1.1内唯一个必须包含在请求内的首部字段。可以解决一个ip对应多个域名的情况。若服务器未设定主机名，就直接发送一个空值。
9. If-xxx 一系列条件，只有结果满足条件才会从服务器返回。
10. Max-Forward 最多转发次数
11. Proxy-Authorization 客户端和代理之间的认证。
12. Range 获取部分资源的范围请求。单位是字节。
13. Referer 请求是从哪个uri过来的（正确拼写应该是Referrer，错误的被沿用了）
14. TE 客户端能处理的传输编码方式及优先级。和Accept-Encoding的区别是用户传输编码
15. User-Agent 创建请求的浏览器和用户代理名称。

#### 响应首部字段

1. Accept-Range 告知客户端服务器是否能处理范围请求。可以时为bytes，不能时为none
2. Age 源服务器在多久前创建了响应，代理创建响应时必须加上该字段。单位是秒
3. ETag 服务器端资源的唯一标识
4. Location 一般在重定向3xx时，返回新的地址
5. Proxy-Authenticate 代理服务器所要求的认证信息。
6. Retry-After 告知客户端多久之后再次发送请求，主要配合503或者3xx使用
7. Server http服务器信息
8. Vary 源服务器对代理服务器传达关于使用缓存的一些控制。比如Vary：Accept—Language，则只对含有相同VAccept—Language的请求使用缓存。
9. WWW-Authenticate 401返回中肯定会带有这个字段，告知客户端需要认证的信息。

#### 实体首部字段

包含在请求报文和响应报文中的实体部门所使用的首部，用于补充内容的更新时间等与实体相关的信息

1. Allow 用于通知客户端能够支持的http方法，当返回时405时，会把支持的http方法写入该字段返回。
2. Content-Encoding 服务器对实体的主体部分内容的编码方式。如gzip，compress等。
3. Content—Language 告知客户端，实体主体使用的自然语言
4. Content-Length 实体主体部分的大小，单位是字节。
5. Content-Location 给出与报文主体部分对应的uri，和Location不同，比如对于使用Accept-Language的请求，当返回的页面和实际请求的对象不同时，Content-Location会写入实际返回的uri
6. Content-MD5 对报文主体部分执行MD5算法，然后再Base64后的结果
7. Content—Range 对于客户端的范围请求，告知客户端返回的是哪一部分，如：Content-Range： bytes 5001-10000/10000
8. Content-Type 实体主体内对象的媒体类型
9. Expires 缓存服务器在Expires字段指定的时间之前，会一直以缓存的内容来响应。超过指定时间后，会转向源服务器请求资源。当通用首部字段Cache-Control有指定的max-age指令时，会优先处理max-age指令。
10. Last-Modified 资源最终被修改的时间。


#### 为cookie服务的首部字段

cookie没有被编入标准化的http／1.1的RFC2016中。现在使用的是netspace的基准上扩展的

1. Set-Cookie 用在响应首部字段中，字段包括

	| 属性 | 说明 |
	|:-----|:---------|
	| NAME=VALUE  | 赋予cookie的名称和其值 |
	| expires=DATE | 指定浏览器可以发送Cookie的有效期，若省略，其有效期为浏览器关闭之前 |
	| path=PATH | 将服务器上的文件目录作为Cookie的适用对象，若省略，默认为文档所在的文件目录，不过这个限制可以避开 |
	| domain=域名 | Cookie适用对象的域名，若省略，默认为创建Cookie的服务器的域名，采用结尾匹配，如指定为a.com, www.a.com, www2.a.com都可以匹配 |
	| secure | 仅在https的通信时才发送Cookie，通过 Set-Cookie: name=value; secure指定 |
	| HttpOnly | 使Cookie不能被JS脚本访问,可以防止XSS，通过 Set-Cookie: name=value; HttpOnly指定 |

2. Cookie 用在请求首部字段中，在请求中包含从服务器接收到的Cookie
	
	
#### 其它首部字段

1. X-Frame-Options 用于控制网站内容在其它web网站的Frame标签内的显示问题，主要为了防止点击劫持（clickjacking）攻击。DENY：拒绝，SAMEORIGIN：仅同源域名下的页面可以加载。
2. X-XSS-Protexiont 属于http响应首部，控制浏览器XSS防护机制的开关，0:XSS过滤设置无效，1:XSS过滤设置有效
3. DNT 属于http请求首部，0:同意被追踪，1:拒绝被追踪
4. P3P 属于http响应首部，利用技术保护个人隐私。
	

### HTTP状态码

100-199 用于指定客户端应相应的某些动作。

200-299 用于表示请求成功。

300-399 用于已经移动的文件并且常被包含在定位头信息中指定新的地址信息。

400-499 用于指出客户端的错误。

500-599 用于支持服务器错误。

* 100 continue  服务器仅接收到部分请求，但是一旦服务器并没有拒绝该请求，客户端应该继续发送其余的请求
* 200 ok 请求已成功，请求所希望的响应头或数据体将随此响应返回。
* 201 Created 请求已经被实现，而且有一个新的资源已经依据请求的需要而建立，且其 URI 已经随Location 头信息返回。
* 202 Accepted 服务器已接受请求，但尚未处理。
* 203  Non-authoritative Information 服务器已成功处理了请求，但返回了可能来自另一来源的信息。
* 204 No Content 服务器已经处理了请求，但没有内容可以返回。浏览器的话显示的页面不更新。
* 206 Partial Content 客户端进行了范围请求，服务器成功执行了，返回由Content-Range指定范围的实体内容
* 300 Multiple Choices 服务器根据请求可执行多种操作。服务器可根据请求者 来选择一项操作，或提供操作列表供其选择。
* 301 Moved Permanently 被请求的资源已永久移动到新位置，并且将来任何对此资源的引用都应该使用本响应返回的若干个 URI 之一
* 302 Found 服务器目前正从不同位置的网页响应请求，但请求者应继续使用原有位置来进行以后的请求。临时重定向。
* 303 See Other 请求的资源对应着另一个URI，应使用GET方法定向获取请求的资源。和302很相似，但是标准中要求302和301都不能将post方法改成get方法的，不过大多数浏览器都会这么做。
* 304 Not Modified 客户端发送附带条件的请求，服务器允许访问资源，但是请求未满足条件表示，服务器资源未改变，可以使用客户端未过期的缓存。客户端发送的附带条件指get方法的请求报文中包含If-Match，If-Modified-Since,If-None-Match,If-Range,If-Unmodified-Since
* 307 Temporary Redirect 请求的资源现在临时从不同的URI 响应请求。由于这样的重定向是临时的，客户端应当继续向原有地址发送以后的请求。和302的区别是，会遵照浏览器的标准，不会从post变成get。
* 400 Bad Request 1、语义有误，当前请求无法被服务器理解。除非进行修改，否则客户端不应该重复提交这个请求。2、请求参数有误。
* 401 Unauthorized 未授权，请求页面需要用户名和密码
* 403 Forbidden 禁止, 服务器拒绝请求
* 404 Not Found 请求失败，请求所希望得到的资源未被在服务器上发现。
* 405 Method Not Allowed 指出请求方法(GET, POST, HEAD, PUT, DELETE, 等)对某些特定的资源不允许使用。
* 406 Not Acceptable 服务器生成的响应客户端无法接受
* 407 Proxy Authentiacation Required 用户必须首先使用代理服务器进行验证
* 408 Request Timeout 请求超出了服务器的等待时间
* 410 Gone 被请求的页面不可用
* 415 Unsupported Media Type  请求的格式不受请求页面的支持
* 417 Excepctation Failed 客户端的请求中加入了Expect字段，服务器无法满足
* 500 Internal Server Error   服务器在执行请求时发生了错误，有可能是存在bug
* 501 Not Implemented 服务器不支持当前请求所需要的某个功能。当服务器无法识别请求的方法，并且无法支持其对任何资源的请求。
* 502 bad gateway，服务器从上游服务器收到一个无效的相应。错误的网关
* 503 Service unavailable，服务器临时过载或宕机
* 504 Gateway timeout 网关超时
* 505 HTTP_VERSION_NOT_SUPPORTED 服务器并不支持在请求中所标明 HTTP 版本。
	
	
	
	
	
	
	
