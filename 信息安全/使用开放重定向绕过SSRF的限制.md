https://portswigger.net/web-security/learning-paths/ssrf-attacks/ssrf-attacks-circumventing-defenses/ssrf/lab-ssrf-filter-bypass-via-open-redirection#


> [!info] 目标
> Lab 要求我们通过 SSRF 访问 `http://192.168.0.12:8080/admin` 来删除 carlos 的账号


在这个 lab 中，我们可以控制一个包里的请求地址
```HTTP
POST /product/stock HTTP/1.1
Host: 0a2c00a403cd7727841486f6008e005e.web-security-academy.net
Connection: keep-alive
Content-Length: 65
sec-ch-ua-platform: "Windows"
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36
sec-ch-ua: "Google Chrome";v="147", "Not.A/Brand";v="8", "Chromium";v="147"
Content-Type: application/x-www-form-urlencoded
sec-ch-ua-mobile: ?0
Accept: */*
Origin: https://0a2c00a403cd7727841486f6008e005e.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a2c00a403cd7727841486f6008e005e.web-security-academy.net/product?productId=3
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9
Cookie: session=o4kLT70fRAUIcFMoIonAD6IxuVqVCZr5

stockApi=%2Fproduct%2Fstock%2Fcheck%3FproductId%3D3%26storeId%3D1
```

问题是，他限制了只能访问本地应用程序，这意味着我们没有办法让他直接访问 `http://192.168.0.12:8080/admin`

搜寻该网页程序的各个接口
![[Pasted image 20260506164756.png]]
在这个“下一个产品”接口这里，具有开放重定向

那么可以利用这个重定向接口绕过限制

![[Pasted image 20260506165254.png]]

---


> [!NOTE] 
> 本案例具备"只能访问网页程序本身"的限制，但使用了网页程序自带的重定向接口绕过了这一限制
