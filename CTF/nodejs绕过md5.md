---
tags:
  - 题目
  - CTF
---

> [!faq] 题目描述
> ```js
> var express = require('express');
> var router = express.Router();
> var crypto = require('crypto');
> 
> function md5(s) {
>   return crypto.createHash('md5')
>     .update(s)
>     .digest('hex');
> }
> 
> /* GET home page. */
> router.get('/', function(req, res, next) {
>   res.type('html'); 
>   var flag='xxxxxxx';
>   var a = req.query.a;
>   var b = req.query.b;
>   if(a && b && a.length===b.length && a!==b && md5(a+flag)===md5(b+flag)){
>   	res.end(flag);
>   }else{
>   	res.render('index',{ msg: 'tql'});
>   }
>   
> });
> 
> module.exports = router;
> ```

---

传参 `?a[]=1&b=1`

类型不同所以 `a!==b` 成立，然后他们都会和 `flag` 拼接成 `1xxxxxxx`，所以md5相同。