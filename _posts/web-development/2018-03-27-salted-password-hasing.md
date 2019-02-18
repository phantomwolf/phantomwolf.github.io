---
layout: post
title: "网站中密码的hash与验证"
date: 2018-03-27 10:47:00 +0800
categories: web-development
---
## 概述

### 密码的hash

1. 客户端(浏览器)通过HTTPS协议向服务器发送密码。
2. 服务器产生一个足够长的随机字符串，作为salt。
3. 服务器将Salt与密码连接起来，形成一个新字符串，并用一种算法将其hash。
4. 将Salt与hash的结果存入数据库。

### 密码的验证

1. 客户端(浏览器)通过HTTPS协议向服务器发送密码。
2. 服务器从数据库取出相应的Salt，与用户发送的密码连接起来，形成一个新字符串并hash。
3. 将hash的结果与数据库中存储的hash结果对比，如果一致，则密码正确；否则，密码错误。

### 注意事项

1. 客户端(浏览器)必须使用HTTPS来发送密码，否则很容易遭到中间人攻击，因为HTTP协议是明文传输的。
2. 服务器必须使用“密码学安全伪随机数生成器(Cryptographically secure pseudorandom number generator, CSPRNG)”来生成随机的Salt，而不能用普通的随机数生成函数。例如在Go中，必须使用crypto/rand模块中的函数。
3. 密码的hash必须使用PBKDF2、bcrypt、scrypt、Argon2等算法，增大破解的难度，而不能使用sha1、sha256、md5等算法。

### 示例代码(Go)：

{% highlight go %}
import "golang.org/x/crypto/bcrypt"
// Encrypt password
password := "foo"
bytes := []byte(password)
encrypted, err := bcrypt.GenerateFromPassword(bytes, cost)
// Verify password
err := bcrypt.CompareHashAndPassword(encrypted, bytes)
{% endhighlight %}

注意：示例中选用的库会自动生成salt。
