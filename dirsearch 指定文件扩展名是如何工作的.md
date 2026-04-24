指定文件扩展名的参数是 `-e`
不指定的时候，dirsearch默认使用 `-e php, asp, aspx, jsp, html, htm`

假如你的字典如下，并使用 `-e php, asp, aspx, jsp, html, htm`
```txt
admin
login.%EXT%
```

那么将会生成请求
```txt
/admin
/login
/login.php
/login.asp
/login.aspx
/login.jsp
/login.html
/login.htm
```