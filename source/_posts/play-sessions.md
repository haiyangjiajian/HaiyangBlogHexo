---
layout: post
title: play登录状态管理
tags: [scala, play]
category: 编程
---

## play登录状态管理

## 目录
- 加密过程
- 模拟登录


#### 加密过程

在play官方[文档](https://www.playframework.com/documentation/2.6.x/ScalaSessionFlash)中有这样的介绍:

It’s important to understand that Session and Flash data are not stored by the server but are added to each subsequent HTTP request, using the cookie mechanism. This means that the data size is very limited (up to 4 KB) and that you can only store string values. The default name for the cookie is PLAY_SESSION. This can be changed by configuring the key session.cookieName in application.conf.

大意是说play通过cookie来保存session的信息，只能保存最大为4k的字符串。默认的cookie名字是PLAY_SESSION，可以在application.conf里面配置。

来看一下这个配置是怎么使用，如何通过这个session来保存用户的登录信息。

在application.conf里有如下配置：

```
# Sessions
session.maxAge = 31536000000
session.cookieName = my_play
application.secret="xxx"
```
application.secret是用来

* Signing session cookies and CSRF tokens
* Built in encryption utilities

在play中一般登录成功后会通过如下方式返回

```
Ok("login success").withSession("userId" -> "555").withCookies(...)
```

在withSession的调用中会把user_id通过application.secret来加密，并保存到cookie中，然后随结果返回。看一下具体执行的play源码

* play.api.mvc.Results.scala中的withSession方法

```Java
def withSession(session: (String, String)*): Result = withSession(Session(session.toMap))

def withSession(session: Session): Result = {
    if (session.isEmpty) discardingCookies(Session.discard) else withCookies(Session.encodeAsCookie(session))
  }
  
```

* play.api.mvc.Http.scala中的encodeAsCookie方法将session转为cookie并签名

```
/**
  * Encodes the data as a `Cookie`.
*/
def encodeAsCookie(data: T): Cookie = {
	val cookie = encode(serialize(data))
  	Cookie(COOKIE_NAME, cookie, maxAge, path, domain, secure, httpOnly)
}

/**
	* Encodes the data as a `String`.
*/
def encode(data: Map[String, String]): String = {
	val encoded = data.map {
	case (k, v) => URLEncoder.encode(k, "UTF-8") + "=" + URLEncoder.encode(v, "UTF-8")}.mkString("&")
	
	//cookie的value是 加密后的串-userId
	if (isSigned)
		Crypto.sign(encoded) + "-" + encoded
	else
       encoded
}
```

* play.api.libs.Crypto.scala使用application.secret对传入的参数进行签名

```
  def sign(message: String): String = {
    sign(message, secret.getBytes("utf-8"))
  }
  
  //最终这个函数返回的就是签名后放在cookie中的userId
  def sign(message: String, key: Array[Byte]): String = {
    val mac = provider.map(p => Mac.getInstance("HmacSHA1", p)).getOrElse(Mac.getInstance("HmacSHA1"))
    mac.init(new SecretKeySpec(key, "HmacSHA1"))
    Codecs.toHexString(mac.doFinal(message.getBytes("utf-8")))
  }
```

#### 模拟登录
了解上面原理后可以手动生成cookie，然后模拟登录
比如上面例子中，最后服务器验证登录的cookie是my_play-userId = 'xxx-555'。只要用上面签名的方法，对userId进行签名就可以生成任意用户的登录cookie。生成的cookies使用可以使用chrome插件 [editThisCookie](https://chrome.google.com/webstore/detail/editthiscookie/fngmhnnpilhplaeedifhccceomclgfbg?hl=zh-CN)

对于开发者来说一定要妥善保存application.secret,最好不要直接保存到application.conf中。可以考虑线上环境作为参数启动程序，或者用线上的配置覆盖application.conf.








