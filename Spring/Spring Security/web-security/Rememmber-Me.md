# Remember-Me
Remeber-Me 或者说 persistent-login authentication(身份认证)
是指网站能够记住会话之间的主体身份.
> 当勾选记住我选项后, 用户完成访问关闭了浏览器.(失去了SessionId)
> 只会重新打开浏览器访问(新的Session), 新的 Session 会自动加载主体身份. 
通常, 这是通过向浏览器发送一个cookie实现的, 
该cookie在以后的会话中被检测到并导致自动登录.


Spring Security提供了两种策略实现记住我功能
1. 使用Hash保证基于cookie的安全性.
2. 使用数据库或其他持久性存储机制在存储生成的令牌.

# Simple Hash-Based Token Approach

> 为了确保能够根据Remember-Me Cookies恢复用户的主体身份.  我们需要用户的标识(通常是 username). 
> 为了确保这个Cookie不会被人利用, 我们还要保存 expirationTime 过期时间
> 光在SetCookies设置max-age值是不够的, 只是浏览器会在时间到了删除这个cookie, 并不是cookie失效了. 
> 如果有人将cookie保存起来, 即使浏览器到期删除了cookie. 但是如果它继续携带cookie访问, 依然可以登录系统.


基于哈希散列保证remember-me cookie token 安全的策略生成的 cookie 格式如下:
```
base64(username + ":" + expirationTime + ":" +
md5Hex(username + ":" + expirationTime + ":" password + ":" + key))

username:          As identifiable to the UserDetailsService
password:          That matches the one in the retrieved UserDetails
expirationTime:    The date and time when the remember-me token expires, expressed in milliseconds
key:               A private key to prevent modification of the remember-me token
```
第一部分用 Base64 编码 username 和 expirationTime, 保证可以通过cookie还原用户登录状态以及cookie过期的事情.
第二部分是 username, password, expireationTime, key 的一个 MD5 消息摘要,
防止remember-me cookies被伪造.
> 重点: 如果是光为了防止伪造, 其实只要加入一个 secret key 就可以了
> 只有服务其中才知道的密钥, 只要密钥不泄露就无法伪造md5结果.
> 不需要把 password 也加入进来.
> 
> 把 password 也一起进行hash有什么好处呢?
> 当用户在记住我登录期间, 修改了账号密码. 那么原来的记住我token就失效了
> 因为在验证后半部分签名时, 验证不通过.(原来的token是用 old password 生成的, 用新的密码自然无法生成一样的)
> 确保了用户在修改密码后, 之前的remember-me cookie token失效.


这个机制有一个问题, 就是这个 cookie 一旦签发, 就被任何用户代理使用使用直到过期.
不过依托上面提到的依赖 password 的签名方式, 一旦用户直到自己的账号被捕获了, 
他们可以通过修改密码来使出现问题的 remember-me token 失效.

# 基于持久化存储机制实现的Remember-Me


[基本思路](https://fishbowl.pastiche.org/2004/01/19/persistent_login_cookie_best_practice)
[需要翻墙, 是对上面的改进](http://jaspan.com/improved_persistent_login_cookie_best_practice)

前提:
1. Cookies很容易受到攻击. 在常见的浏览器cookie盗窃漏洞和跨站点脚本攻击之间, 我们必须接受cookie并不安全

2. 持久登录cookie本身具有足够的身份验证才能访问网站. 它们等同于将有效的用户名和密码合并为一个

3. 用户重用密码. 因此, 任何您可以从中恢复用户密码的登录cookie所带来的潜在危害比您无法提供的登录cookie所具有的危害可能性要大得多.

4. 将持久性cookie绑定到特定IP地址使得它们在许多常见情况下不是特别持久

5. 用户可能希望同时在不同机器上的多个Web浏览器上拥有持久性Cookie


这个策略是为了解决之前简单的策略的安全问题: 
当用户的 remember-me cookie token被盗取时, 在token过期之前攻击者都通过其可以访问系统.
> 最简单的方式, 我们可以采用和 IP 地址绑定的方式,
> 如果请求者携带 remember-me token 但是 IP 和登录时的不同, 就无视这个token.
> 这样就避免了 token 被盗后可以使用的问题. (攻击者的IP和用户IP不同, token 不生效)
> 不过这个简单的方法是无法投入使用的, 因为用户的IP通常会变化.
> 用户在家中可能是一个IP, 到户外连接4G, 或者什么公共场所的Wifi都会导致IP变化.
> 从而使记住我功能失效. 用户体验极差.
>
> 这个问题本质是, remember-me token 的持续时间长, 并且不会*变化*.
> 当然我们不能让时间变短(只能记住几小时的记住我每什么意义), 
> 但是我们可以让其发生变化, 
> 比如说每次如果用户通过 remember-me token 登录后, 重新生成新的token并设置cookie.
> 这样如果用户的某次 remember-me token 被盗了, 
> 只要用户下次通过这个token登录后, 新的token就会生成, 被盗的token就失效了. 
> 
> 当然这里有个小问题, 就是如果token_0被盗后, 攻击者依旧可以通过token_0直接访问提供系统,
> 并且新的token_1也会被攻击者获取到. (意味着攻击者可以一直访问, 因为一直可以获取下一个新的token)
> 直到用户尝试使用旧的 token_0 登录系统时发现token失效,
> 重新通过交互式身份验证登陆系统, 触发新的 remember-me token 的生成时攻击者获取到的 token 才会无效.
> 这个问题是第二篇文章主要改进的.(后面再讲)
> 
> 还有一个问题比较严重, 就是无法做到, 不同的机器上的多个Web浏览器上拥有持久性cookie.
> 说人话就是, 只能在一台机器的浏览器上记住我, 当另一台机器进行记住我时, 原来的也就失效了.
> 原因也就是之前使token经常变化导致的.(每次登陆都会变化, 只有一个最新的token是有效的, 所以也只能再一个地方记住我, 其他会被挤下来)
> 那么怎么解决呢? 这就用到了标题里提到的技术了(持久化存储技术).
> 
> 之前的问题本质上是: 每次都会变化token, 一个用户只会有一个token, 所以只能在一台机器上记住我.
> 如果我们允许用户同时有多个 remember-me token 的话, 就解决了问题.
> 为了允许用户有多个token, 我们需要维护token的列表.
> 下面是具体方法:
> remember-me token 可以采用 username + 合适大小的随机数
> username 是为了能够恢复用户的登陆状态.
> 随机数是为了标识这个用户的这个token(或者说是用来标识不同的机器的?).
> 然后我们在数据库中保存 number -> username 的关联表.
> 允许一个用户有多个token, 即可以存在多个 number 关联同一个 username.
>
> 在用户通过账号密码登陆并勾选了记住我选项后, 为其生成一个 username + 随机数 形式的token.
> 并保存 number -> username 映射关系到数据库.
> 以后每次通过记住我服务登陆时, 需要判断随机数是否有效, 并且每次更新token就是改变随机数的值.
> 这样就允许用户在多个机器上登陆, 并且记住我.
> 
> 当用户在机器A上登陆后, 系统生成一个 username + 232 的token.
> 然后用户在机器B上登陆后, 提供生成一个 username + 123 的token.
> 这两个token都是有效的, 可以在数据库中查询到 number -> username映射, 并且是独立的.
> 当然有个小问题, 就是每次重复生成随机数时需要避免重复
> *注意, 这里的重复不是 123 -> wkunc; 123 -> bill 这样的重复, 这样的重复时允许的.
> 不允许重复是如果已经有一个 123 -> wkunc在生成新的token时需要避免生成一样的 123 -> wkunc
> 在一个新的token生成时需要避免和之前的token的随机数相同.否则它们就是代表同一token.*
> 不过考虑一个合适的随机数大小, 就可以忽略这个问题. 比如 0~~128 的范围.
> 通常我们对一个用户能同时记住我的数量也有限制(就是持久化token的数量限制)
> 如爱奇艺限制了 5 个.
> 在最多只有5个被占用的随机值时, 生成一个随机数和之前的相同的概率只有 5/128, 可以忽略不计.
> 当用户注销某个机器的登陆状态时, 意味着对应的随机数也就无效了(删除数据库中的记录)
> 然后还有有一个接口允许用户同时注销所有的 remember-me token 以防万一.
> 最后需要定期数据库中存在时间超过一定长度的关联, 比如说一个月(实现 remember-me token 的主动失效)
> 
> 最后给出权限意见即记住我方式登陆的用户无法访问一些比较敏感的信息
> 比如: 1. 修改用户密码 2. 修改用户邮箱 3. 访问用户钱包(财务相关) 4. 用户地址
> 
> 以上就是我对第一份参考博客的理解. 接下来解释第二篇博客对这个方式的改进(修复之前提到的问题, )

