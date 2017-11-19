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

`spawn()` 返回一个`ChildProcess`, 他的 `stdin`, `stdout`, and `stderr` 属性都是 `Streams`, 可用于读写数据, 使用 `pipe()` 输出到标准输出流.

### 从 EventEmitter 中获取数据
`Stream` 是从 `EventEmitter` 继承而来.

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
  let ls = spawn('ls', ['-lh', resolve(__dirname, filename)]);
  let output = '';

  ls.stdout.on('data', function (chunk) {
    output += chunk.toString();
  });

  ls.on('close', function () {
    let parts = output.split(/\s+/);
    console.dir([parts[0], parts[4], parts[8]]);
  });
});
console.log("Now watching " + filename + " for changes...");
```

以上, 我们给添加了一个事件监听, 一个监听事件是一个回调函数, 当特定的事件类型下发, 回调函数就会被调用.

Buffer 用于在 node 中展示二进制数据. 调用 `.toString()` 用 node 的默认编码方式(UTF-8) 将 buffer 内容转换为 JavaScript 的 String.

### 异步读写文件

读文件:
```js
const fs = require('fs');
const resolve = require('path').resolve;

fs.readFile(resolve(__dirname, 'target.txt'), function (err, data) {
  console.log(err);
  if (err) {
    throw err;
  }
  console.log(data.toString());
});
```

an uncaught exception in Node 会因为脱离 event loop 而停止程序.

第二个参数 data 是一个 buffer

### 创建读写流
```js
const
  fs = require('fs'),
  stream = fs.createReadStream(process.argv[2]);
stream.on('data', function (chunk) {
  process.stdout.write(chunk);
});
stream.on('error', function (err) {
  process.stderr.write("ERROR: " + err.message + "\n");
});
```

### Blocking the Event Loop with Synchronous File Access
如果node进程被阻塞, 意味着node不会执行其他任何代码, 不会触发回调函数, 不会处理事件, 不会接受任何连接. 

### node 程序的两个阶段
程序的初始化时期, 可以进行文件的同步读取.

第二个时期是操作时期, 程序进入 event loop, 在这个时期 **绝对不能使用** 同步文件读取.

`require()` 函数是同步进行的. 

## Networking with Sockets

### Listening for Socket Connections
网络服务做两件事: 连接两端, 传输信息.

#### Binding a Server to a TCP Port
TCP socket 连接包含两个端点. 一端绑定到数值型端口上, 另一端连接到一个端口.

在node中, 绑定了连接操作由 `net` 模块提供.

绑定一个TCP端口用于监听连接:
```js
"use strict";
const
  net = require('net'),
  server = net.createServer(function (connection) {
    // use connection object for data transfer
  });
server.listen(5432);
```

connection 参数是一个 Socket 对象, 可以用于接收和发送数据.

#### Writing Data to a Socket
```js
'use strict';
const
  fs = require('fs'),
  net = require('net'),
  filename = process.argv[2],
  server = net.createServer(function (connection) {
    // reporting
    console.log('Subscriber connected.');
    connection.write("Now watching '" + filename + "' for changes...\n");
    // watcher setup
    let watcher = fs.watch(filename, function () {
      connection.write("File '" + filename + "' changed: " + Date.now() + "\n");
    });

    // cleanup
    connection.on('close', function () {
      console.log('Subscriber disconnected.');
      watcher.close();
    });
  });
if (!filename) {
  throw Error('No target filename was specified.');
}
server.listen(5432, function () {
  console.log('Listening for subscribers...');
});
```

在回调函数中使用 `connection.write` 给客户端发送文件变化.

#### Connecting to a TCP Socket Server with Telnet

通过telnet命令, 连接TCP socket 服务器

```js
'use strict';
const
  fs = require('fs'),
  net = require('net'),
  resolve = require('path').resolve,
  filename = resolve(__dirname, process.argv[2]),
  server = net.createServer(function (connection) {
    // reporting
    console.log('Subscriber connected.');
    connection.write("Now watching '" + filename + "' for changes...\n");
    // watcher setup
    let watcher = fs.watch(filename, function () {
      connection.write("File '" + filename + "' changed: " + Date.now() + "\n");
    });

    // cleanup
    connection.on('close', function () {
      console.log('Subscriber disconnected.');
      watcher.close();
    });
  });
if (!filename) {
  throw Error('No target filename was specified.');
}
server.listen(5432, function () {
  console.log('Listening for subscribers...');
});
```
3个终端:
1. node test.js target.txt
2. telnet localhost 5432
3. vi target.txt

通过 `Ctrl-]` 和 `Ctrl-C` kill telnet, 通过 `Ctrl-C` kill node.

![img](http://oe3zwqfm1.bkt.clouddn.com/3D8A43ED5DAA033EA401903A2E1C10A0.jpg)

TCP sockets 对于网络上的两台计算机之间的交流很有用. 在同一台计算机上的交流, Unix sockets 更高效, `net` 模块也提供这种方式.

#### Listening on Unix Sockets

Unix sockets 只能在 Unix-like 环境下工作.

修改我们之前的代码成:
```js
'use strict';
const
  fs = require('fs'),
  net = require('net'),
  resolve = require('path').resolve,
  filename = resolve(__dirname, process.argv[2]),
  server = net.createServer(function (connection) {
    // reporting
    console.log('Subscriber connected.');
    connection.write("Now watching '" + filename + "' for changes...\n");
    // watcher setup
    let watcher = fs.watch(filename, function () {
      connection.write("File '" + filename + "' changed: " + Date.now() + "\n");
    });

    // cleanup
    connection.on('close', function () {
      console.log('Subscriber disconnected.');
      watcher.close();
    });
  });
if (!filename) {
  throw Error('No target filename was specified.');
}
server.listen('/tmp/watcher.sock', function() {
  console.log('Listening for subscribers...');
});
```

我们此时需要 `nc` 而不是 `telnet`, `nc` 是 `netcat` 的简称, 用于 TCP/UDP socket 也支持 Unix sockets.

```shell
$ nc -U /tmp/watcher.sock
```

Unix sockets 比 TCP sockets 更快, 因为他不需要调用网络硬件, 不过只能在本地使用.

### Implementing a Messaging Protocol
`protocol(协议)` 是端之间进行交流的一系列的规则.

这里我们会使用基于 `JSON` 的协议.

#### 把消息用JSON序列化

我们需要把之前的消息转换为JSON格式, 比如:

字符串 `Now watching target.txt for changes...` -> `{"type":"watching","file":"target.txt"}`

字符串 `File ’target.txt’ changed: Sat Jan 12 2013 12:35:52 GMT-0500 (EST)` -> `{"type":"changed","file":"target.txt","timestamp":1358175733785}`

#### Switching to JSON Messages

我们还需要创建client程序, 接受和解释messages.

