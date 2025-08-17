## web安全
### XSS（跨站脚本攻击 Cross Site Scripting）
- 案例
	url注入恶意代码
	数据库存入恶意代码
- 解决
	- 将cookie等敏感信息设为httpOnly，禁止js通过document.cookie获得；
	- 对输入做严格校验，过滤不合法输入；
	- 过滤一些不必要的标签，<iframe>,<script>等；
	- 转义特殊符号；
### 跨站请求伪造（CSRF Cross Site Request Forgery）
- 概念
  引用用户打开黑客网站，在黑客的网站中，利用用户登录状态发起跨站请求；
  -必要条件
  - 目标站点一定要有CSRF漏洞；
  - 用户要登录过目标站点，并在浏览器上保持改站点的登录信息；
  - 需要用户打开一个第三方的站点，如黑客站点；

- 解决
  - 验证请求来源；
  	判断请求的Origin和Referer；
  		Referer是http请求头中的一个字段，记录该请求的来源地址；
  		Origin属性只包含域名信息，并没有包含具体的url路径；
  		服务端会优先判断Origin，如果不存在，再根据实际情况使用Referer的值；
  - 利用cookie的SameSite属性
  	选项：Strict，Lax，None；
  	- 区别
  		- Strict，浏览器会完全禁止第三方Cookie；
  		- Lax，从第三方链接打开 和 从第三方站点提交Get方式的表单，都会携带cookie；
  		在第三方站点使用Post，或通img，iframe等标签加载url，不带cookie；
  - 使用token验证
  	所有请求的身份信息判断使用token来验证
  		token工作原理：
  		todo

- ### CSP

  - 内容安全策略，在HTML的meta标签或者服务端返回的Content-Secrity-Policy头中进行设置
  - 可以指定资源的请求域、资源的加载方式等

### 防重放

