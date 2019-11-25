# CSRF (Cross-site request forgery)
跨站请求伪造

CSRF攻击是利用用户身份操作用户操作用户的一种攻击方式,
主要原因时利用用户登陆后服务器设置的Cookie进行伪造.

就是伪造一个奇怪的链接, 引诱用户点击访问.
链接指向攻击目标网站, 当用户点击时进行访问,
并且浏览器会带上对应网站的cookies. 如果网站使用cookies表明用户状态,
那么这次操作就会被认为是一个登陆用户进行的操作.
