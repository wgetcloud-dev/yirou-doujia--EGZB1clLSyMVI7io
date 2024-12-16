
### 服务端如何验证客户端已经登录？


在用户成功登录后，服务端会发放一个凭证。之后，客户端的每次请求都需要携带该凭证，服务端通过验证凭证的有效性来判断用户是否已登录，并处理请求。


以下是 **Session** 和 **JWT** 在这方面的不同之处：


**1\. 凭证的内容是什么？**


* **Session**：凭证是一个简单的 **ID 字符串**，用于映射服务端存储的会话信息。




```
JSESSIONID=8C3C44A3A0B522F1B93D3F8C4F17F2E7; Path=/your-web-app; HttpOnly
```


* **JWT**：凭证是一个 **自包含** 的令牌，解码后可以直接获得会话信息（如用户信息、权限等）。




```
//JWT token格式
..

//JWT 案例Token
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.S5rf1jOSGbHkGBv1buVpzCHYtoJXnJkK9J9M2yExhms

//JWT 案例token 各部分解码
{"alg":"HS256","typ":"JWT"}.{"userId":"1234567890","name":"John Doe","iat":1516239022}.S5rf1jOSGbHkGBv1buVpzCHYtoJXnJkK9J9M2yExhms

//注意：JWT并不是加密数据，只是提供了签名防止内容被篡改。Header和Payload都是明文。
```


**2\. 凭证如何传输？**


* **Session**：服务端通过 **Set\-Cookie** 方式将凭证发送给客户端，客户端将凭证存储在 cookie 中，并在每次请求中通过 **Cookie** 头部将其发送回服务端。
* **JWT**：服务端将凭证返回给客户端，客户端存储在 **localStorage** 或 **sessionStorage** 中。后续请求中，客户端通过 **Authorization** 头部将 JWT 传递给服务端。


**3\. 服务端如何检测凭证？**


* **Session**：服务端根据接收到的 **sessionId** 查询会话信息，若会话存在则验证通过，服务端继续处理请求。
* **JWT**：服务端验证 JWT 中的签名，签名通过表明凭证未被篡改，且确实是由服务端签发的。（二开应用记得修改JWT签名密钥，否则其他系统的JWT也能访问你的系统）


**4\. 认证信息存储在哪里？**


* **Session**：认证信息存储在服务端内存中，或存储在共享的内存系统（如 Redis）中。
* **JWT**：认证信息存储在客户端的凭证中，JWT 令牌本身包含了所有认证信息。




---


### 关于水平扩展


**Session 的缺点**


当系统只有单个服务时，会话信息可以存储在应用内部。但一旦服务拆分为多个节点，会话信息需要迁移到共享存储（如 Redis）。此时，所有服务的请求都必须访问 Redis。如果 Redis 故障或宕机，服务将无法正常工作，直到 Redis 恢复。


**JWT 的优势**


JWT 的一个主要优势是其 **自包含** 特性，所有认证信息都存储在客户端的凭证中。由于 JWT 的签名是通过服务端的密钥进行验证，服务端无需访问共享存储系统（如 Redis），每个服务节点都可以独立地验证凭证。这使得 JWT 在分布式架构中更具优势，避免了 Redis 故障时的单点风险。




---


### 关于会话控制


**Session 的优点**


由于会话信息存储在服务端，服务端可以方便地管理会话。例如，若需要 **踢人**，可以直接删除 Redis 或内存中的会话信息，之后客户端的请求会被拒绝，用户需重新登录。


**JWT 的缺点**


由于 JWT 在客户端存储，服务端无法直接删除客户端的凭证。如果需要实现踢人操作，服务端必须记录用户签发的 token ID，并通过 **黑名单** 来阻止客户端使用已失效的凭证。


有时候有人担心使用黑名单会与 Session 的方式相似，仍然依赖于一个中心数据存储。但可以通过以下方式解决：


* 中心数据（Redis）存储用户签发的情况，并将黑名单推送到各个应用。
* 每个应用本地保存黑名单副本，这样就不需要频繁查询中心存储。


此外，JWT 有过期时间，黑名单可以设置相同的过期时间，避免过多的数据存储。并且，踢人操作并不频繁，所以黑名单的数据量不会很大。


如果不需要“立即”封禁 JWT，可以考虑使用 **短生命周期的 JWT** 结合 **刷新机制**。这种方式也非常常见。即服务端返回两个 token：


* **access\_token** （JWT）
* **refresh\_token** （通常是一个 UUID，服务端保存该 key 到 Redis 中，value 为用户信息）


客户端使用 **refresh\_token** 生成新的 **access\_token**。




```
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.S5rf1jOSGbHkGBv1buVpzCHYtoJXnJkK9J9M2yExhms
"
    "refresh_token": "c0cdf579-54d5-46fc-8e1b-bd42bd01b556"
}
```


 




---


### 关于会话存储的内容


* **Session**：由于会话信息存储在服务端，服务端可以在会话中存储一些经常访问的数据。
* **JWT**：JWT 的会话内容是 **明文** 的，虽然签名可以防止篡改，但依然可能被读取，因此敏感数据不应放在 JWT 中。另外，JWT 是每次请求都携带的，数据过大会占用带宽，因此不适合存储过多的信息。




---


### 关于页面路由控制


* **JWT** 非常适合 **SPA**（单页面应用），因为页面路由控制通常由前端负责，后端仅通过 JWT 验证数据访问权限。
* 对于 **MPA**（多页面应用），使用 JWT 控制页面访问时会遇到困难。因为在浏览器行为（如页面刷新）中，JWT 需要通过 **Authorization** 头部传递，但刷新页面时，浏览器行为不能直接携带该信息。


解决方案：后端可以在用户登录时不仅在 **body** 中返回 JWT，还通过 **Set\-Cookie** 传递 token。这样客户端请求时会自动携带 JWT，后端校验的时候可以从 **Cookie** 或 **Authorization** 头部两个数据源获取 token。




---


### 总结


* **Session**：将会话信息存储在服务端，凭证是会话 ID，适用于单一应用场景或需要共享内存（如 Redis）的环境，但依赖于共享存储系统。
* **JWT**：凭证包含所有认证信息，并通过签名确保其完整性，适用于分布式架构，且无需依赖外部存储系统。


在拆分为多个服务的分布式系统中，JWT 通过自证机制避免了单点故障的风险，而 Session 则需要一个可靠的存储系统来共享会话信息。具体选择哪种方式，应根据系统的规模、需求和容错能力来决定。


 本博客参考[FlowerCloud机场订阅官网](https://hanlianfangzhi.com)。转载请注明出处！
