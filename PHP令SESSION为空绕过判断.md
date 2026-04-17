---
tags:
  - CTF
  - ctfshow
  - 题目
---
```php
<?php
        function replaceSpecialChar($strParam){
             $regex = "/(select|from|where|join|sleep|and|\s|union|,)/i";
             return preg_replace($regex,"",$strParam);
        }
        if(strlen($password)!=strlen(replaceSpecialChar($password))){
            die("sql inject error");
        }
        if($password==$_SESSION['password']){
            echo $flag;
        }else{
            echo "error";
        }
    ?>
```

PHP会给浏览器分配一个 `SESSIONID` ，这将存在cookie里，我们将其清空或者改成一个不存在的session

这样session就会清空