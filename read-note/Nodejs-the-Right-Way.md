# Node.js the Right Way

**当我们编写node代码的时候, 经常需要处理error或者exception, 即使是将它们抛出.**

## Getting Started

nodejs的应用范围:
![img](http://oe3zwqfm1.bkt.clouddn.com/F1530CCC-CAFF-40DE-875D-942D33B8BF3C.png)

event loop 工作原理:
![img](http://oe3zwqfm1.bkt.clouddn.com/D0ACC7E48639932C0EF0D031ED528B00.jpg)

一个事件上可以添加很多回调, 不过一个时间里只能执行一个回调.

nodejs 单线程, 事件驱动的, 拥有非阻塞api

nodejs 使用C写的事件轮询调度工作, 不过在JavaScript的环境下执行代码

## Wrangling the File System

### 监听文件的变化.

node 的模块的实现是基于 CommonJS 模块规范的.

例子:
```js
const fs = require('fs');
const path = require('path');

fs.watch(path.resolve(__dirname, 'target.txt'), function () {
  console.log("File 'target.txt' just changed!");
});
console.log("Now watching target.txt for changes...");
```
在这个例子中, node 加载脚本, 执行完最后一行代码, 输出 `Now watching...` 之后, 因为调用了 `call()`, 它知道还有一些事没完成, `fs` 模块监听文件的变化.

如果一个异常被抛出, 没有被捕获, 进程会退出.

### 读取命令行参数
```js
const
  fs = require('fs'),
  filename = process.argv[2],
  resolve = require('path').resolve;

if (!filename) {
  throw Error("A file to watch must be specified!");
}
fs.watch(resolve(__dirname, filename), function () {
  console.log("File " + filename + " just changed!");
});
console.log("Now watching " + filename + " for changes...");
```
**在JavaScript里, 函数是一等公民, 我们可以将函数赋给变量**

```js
"use strict";
const
  fs = require('fs'),
  spawn = require('child_process').spawn,
  filename = process.argv[2],
  resolve = require('path').resolve;
if (!filename) {
  throw Error("A file to watch must be specified!");
}
fs.watch(resolve(__dirname, filename), function () {
  console.log('aaaa');
  let ls = spawn('ls', ['-lh', resolve(__dirname, filename)]);
  ls.stdout.pipe(process.stdout);
});
console.log("Now watching " + filename + " for changes...");
```

`spawn()` 返回一个`ChildProcess`, 他的 `stdin`, `stdout`, and `stderr` 属性都是 `Streams`, 可用于读写数据.


