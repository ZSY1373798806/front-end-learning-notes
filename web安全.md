## Web 安全

### 1. XSS（跨站脚本攻击 Cross-Site Scripting）

**概念**
 攻击者通过向网页注入恶意脚本，获取用户信息、劫持会话、篡改页面内容等。

**典型案例**

- URL 注入恶意代码
- 将恶意代码存入数据库（Stored XSS）

**防护措施**

1. **Cookie安全设置**
   - 设置 `HttpOnly` 属性，防止 JS 访问 `document.cookie`。
2. **输入校验与过滤**
   - 严格验证用户输入，过滤非法字符或不必要的标签（如 `<script>`, `<iframe>`）。
3. **转义特殊符号**
   - 将 `<`、`>`、`"`、`'` 等字符转义，避免被浏览器解析为 HTML/JS。

------

### 2. CSRF（跨站请求伪造 Cross-Site Request Forgery）

**概念**
 攻击者诱导已登录用户访问恶意网站，从而以用户身份发起对目标网站的请求。

**发生条件**

1. 目标站存在 CSRF 漏洞
2. 用户已登录目标站，并且浏览器中保留了登录信息（Cookie）
3. 用户访问了第三方恶意站点

**防护措施**

1. **验证请求来源**

   - 检查 `Origin` 或 `Referer` 请求头
     - `Origin`：只包含域名信息
     - `Referer`：包含完整 URL
     - 优先使用 `Origin`，不存在时再使用 `Referer`

2. **Cookie SameSite 属性**

   | 选项   | 行为                                                         |
   | ------ | ------------------------------------------------------------ |
   | Strict | 浏览器完全禁止第三方 Cookie                                  |
   | Lax    | 第三方链接打开或提交 GET 表单会携带 Cookie，但 POST 或 img/iframe 请求不带 Cookie |
   | None   | 第三方请求始终携带 Cookie（必须 HTTPS）                      |

3. **Token 验证**

   - 在请求中加入防伪 Token（如 CSRF Token）
   - **工作原理**：
     1. 服务端生成唯一 Token，并在页面中或响应中传给客户端
     2. 客户端每次请求必须携带该 Token
     3. 服务端验证 Token 与用户会话是否匹配，不匹配则拒绝请求

------

### 3. CSP（Content Security Policy 内容安全策略）

**概念**
 通过 HTTP 响应头或 HTML `<meta>` 标签定义允许加载的资源来源和类型，防止 XSS、数据注入等攻击。

**示例**

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.cdn.com
```

- 限制脚本、样式、图片等资源只从可信域加载
- 可以禁止内联脚本执行

------

### 4. 防重放攻击（Replay Attack）

**概念**
 攻击者截获合法请求并重复发送，从而达到欺骗或非法操作的目的。

**防护措施**

1. **时间戳限制**
   - 请求携带时间戳，服务器验证请求是否过期
2. **一次性 Token（Nonce）**
   - 每次请求生成唯一标识，只能使用一次
3. **数字签名 / HMAC**
   - 对请求参数和时间戳签名，防止被篡改或重放
