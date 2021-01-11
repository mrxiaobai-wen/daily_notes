> 因为前一段时间做了一个系统持续操作期间自动刷新token有效性的需求，然后就想着找一个空闲时间总结一下JWT，所以今天就简单的记录一下自己了解的内容。

***

## JWT是什么

1. JWT全称是JSON Web Token，是一个开放标准，它定义了一种紧凑的、自包含的结构，可用于在服务之间信息传递和授权认证。

+ 信息传递：通过签名，可以确保传递的信息不被篡改且不是伪造的。
+ 授权认证：通过JWT生成的token信息进行验签，确保是自己服务签发的token信息。

2. JWT的格式：

JWT由三部分组成，各部分之间使用[.]进行连接。各部分分别是：Header（头信息）、Payload（载荷）和Signature（签名）。

其格式如下：

~~~
header.payload.signature
~~~

3. 使用场景

前端页面点击登录，后端验证用户名密码通过后，使用JWT生成token信息返回给前端，前端使用登录token请求各个接口。

***

## JWT使用示例

首先引入JWT jar包如下：

~~~xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.6.0</version>
</dependency>
~~~

Java示例代码如下：

~~~java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jws;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import java.util.*;

public class JWTDemo {

    public static void main(String[] args) {
        String issue = "ISSUE_CESHI";
        // 签名算法
        SignatureAlgorithm algorithm = SignatureAlgorithm.HS256;
        // 签名秘钥
        String secret = "CESHI_SECRET";
        int timeout = 60 * 60 * 6;

        long currentTime = System.currentTimeMillis();
        Map<String,Object> map = new HashMap<>();
        map.put("userId","10");
        map.put("userName","张三");

        String token = Jwts.builder()
                .signWith(algorithm, secret)
                .setClaims(map)
                .setId(UUID.randomUUID().toString())
                .setIssuedAt(new Date(currentTime))
                .setIssuer(issue)
                .setExpiration(new Date(currentTime + timeout * 1000))
                .compact();
        System.out.println(token);

        Jws<Claims> parseResult3 = Jwts.parser().setSigningKey(secret).parseClaimsJws(token);
    }
}
~~~

上面演示代码生成的token信息如下：

~~~
eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJU1NVRV9DRVNISSIsInVzZXJOYW1lIjoi5byg5LiJIiwiZXhwIjoxNjEwMzgzMzY0LCJ1c2VySWQiOiIxMCIsImlhdCI6MTYxMDM2MTc2NCwianRpIjoiY2QwNDM1ODctNGZmNS00OGY4LTk2YjUtNTA1ZTc4MzFkMGNjIn0.b_Fr8hpTC5nlwR4NikbEg1WpDMya2Gr5fCfNTV0iCOQ
~~~

然后将生成的token进行解析得到如下内容：

![image-20210111184932241](JWT初识记录.assets/image-20210111184932241.png)

可以看到载荷里面有我们设置的userId和userName信息。

同时，JWT规定了7个官方的字段，如下：

> - iss (issuer)：签发人
> - exp (expiration time)：过期时间
> - sub (subject)：主题
> - aud (audience)：受众
> - nbf (Not Before)：生效时间
> - iat (Issued At)：签发时间
> - jti (JWT ID)：编号

需要特别注意的是，JWT的header和body部分都是用Base64进行编码的，本身不具有加密属性，所以在body中不能存储敏感信息。

***

## 系统持续操作自动刷新token

JWT的一大优点就是token信息存储在客户端，服务端不用存储对应的授权信息。因此，它的一大缺点就是一旦token颁发了就没办法进行撤销，只有等待token自动失效。

如果想要实现token失效前撤销，那么只能借助其他手段，比如使用redis缓存，token验证除了JWT本身的验签外，还需要判断redis缓存信息，但是这样做有违JWT的设计本意。

说说在项目中遇到的需求：就是在用户持续间隔时间内操作系统，那么系统内部应该自动刷新token信息，而不是在固定时间点强行要求重新登录。比如token颁发的有效期是5个小时，然后用户一直操作到4小时59分59秒，然后在5小时0分1秒的时候提交一个表格，这时如果因为token失效直接跳转到登录页面，这样的用户体验感是极为糟糕的。

所以为了解决上面的场景，在写这个需求的时候，特地调研了一下各种方案，最后综合了一下，决定采用redis实现。方案如下：

颁发一个足够时长的token（比如15天），然后存储在redis中，redis中设置短的有效期，比如6小时。从而实现在6小时内有效操作免登录。





