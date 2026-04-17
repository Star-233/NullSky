[web5](https://ctf.show/challenges#web5-10)

```php
<?php
error_reporting(0);
    
?>
<html lang="zh-CN">

<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta name="viewport" content="width=device-width, minimum-scale=1.0, maximum-scale=1.0, initial-scale=1.0" />
    <title>ctf.show_web5</title>
</head>
<body>
    <center>
    <h2>ctf.show_web5</h2>
    <hr>
    <h3>
    </center>
    <?php
        $flag="";
        $v1=$_GET['v1'];
        $v2=$_GET['v2'];
        if(isset($v1) && isset($v2)){
            if(!ctype_alpha($v1)){
                die("v1 error");
            }
            if(!is_numeric($v2)){
                die("v2 error");
            }
            if(md5($v1)==md5($v2)){
                echo $flag;
            }
        }else{
        
            echo "where is flag?";
        }
    ?>

</body>
</html>
```

字母和数字的md5怎么相等？

这里不是强判断而是弱判断，可以考虑 `0e` [[md5碰撞]]
`?v1=QNKCDZO&v2=240610708`
