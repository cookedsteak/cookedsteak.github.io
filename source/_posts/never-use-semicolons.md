---
layout: post
title: 放弃分号呗
category: 技术
keywords: javascript,编程,编码规范,前端,js
comments: true
---

[原文戳这里](https://feross.org/never-use-semicolons/)
本篇是对FEROSS写的一篇文章进行的翻译整理。

## ASI

所有写javascript的程序员都应该知道 ASI。
那ASI是什么鬼呢？
- Automatic Semicolon Insertion
自动分号补全机制

这里是ECMA的诠释 [有兴趣看这里](http://www.ecma-international.org/ecma-262/6.0/index.html#sec-automatic-semicolon-insertion)

> Certain ECMAScript statements (empty statement, let, const, import, and export declarations, variable statement, expression statement, debugger statement, continue statement, break statement, return statement, and throw statement) must be terminated with semicolons. Such semicolons may always appear explicitly in the source text. For convenience, however, such semicolons may be omitted from the source text in certain situations. These situations are described by saying that semicolons are automatically inserted into the source code token stream in those situations.

按照ECMAScript标准，一些特定语句（statement) 必须以分号结尾。分号代表这段语句的终止。但是有时候为了方便，这些分号是有可以省略的。这种情况下解释器会自己判断语句该在哪里终止。这种行为被叫做 “自动插入分号”，简称 ASI (Automatic Semicolon Insertion) 。实际上分号并没有真的被插入，这只是个便于解释的形象说法。

## 举个例子

```
function foo () {
  return
    {
      bar: 1,
      baz: 2
    };
}
```
你认为这样很完美，特别是加了一个分号。但其实呢，这段代码会被自动翻译成：
```
function foo () {
  return; // <– ASI adds a semicolon here. You now have a bug!
    {
      bar: 1,
      baz: 2
    };
}
```
所以有些时候不是你觉得加了分号就有规范有用的。
ASI是ECMAScript的一个规范，所以几乎所有浏览器都有相同的标准。
目前看来，我们可以用一个linter来提示ASI的一些异常行为。
最有名的应该是[ESLint](https://cn.eslint.org/)了
这些属于js的ide插件范畴，大家各取所需。

按照正常的写法，是很难被asi搞错的，反倒是有时候加了分号则成了一个错。
```
function foo () {
  return 42; // ok
};  
// =====
var foo = function () {
}; // ok
// =====
class Foo {
  constructor () {
    if (baz) {
      return 42; // ok
    };           // <– AVOID!
    return 12;   // ok
  };             // <– AVOID!
};               // <– AVOID!
```
如上，有很多类似的例子。
如果你从不使用分号，请记住以下规则：
- 请不要以 `[` `(` 和 ` 作为行开头

如果一定要，请在[]前增加一个分号
```
;[1, 2, 3].forEach(bar)
```
但其实我们有更好的更清晰的写法：
```
const nums = [1, 2, 3]
nums.forEach(bar)
```

也许是因为个人习惯，我比较倾向于不写分号，因为这样让代码更加简洁，而且有时候会强迫你让代码缩进变得可读性更高。

## 参考
1. https://feross.org/never-use-semicolons/
2. http://www.ecma-international.org/ecma-262/6.0/index.html#sec-automatic-semicolon-insertion
3. https://segmentfault.com/a/1190000004548664