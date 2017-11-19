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

### Creating Socket Client Connections

```js
"use strict";
const
  net = require('net'),
  client = net.connect({
    port: 5432
  });
client.on('data', function (data) {
  let message = JSON.parse(data);
  if (message.type === 'watching') {
    console.log("Now watching: " + message.file);
  } else if (message.type === 'changed') {
    let date = new Date(message.timestamp);
    console.log("File '" + message.file + "' changed at " + date);
  } else {
    throw Error("Unrecognized message type: " + message.type);
  }
});
```

`client` 对象是一个 Socket, 就好像 `server` 中的 `connection`.

### Testing Network Application Functionality

我们需要对 server 和 client 进行功能测试.

#### Understanding the Message-Boundary Problem

最好的情况下, 消息会一次性全部抵达. 不过有时候, 消息会分批抵达, 分块在 `data` 事件中.

在上面的 client 代码中, `let message = JSON.parse(data);` 会解析一个data对象, 但是如果接收到的data对象是一段一段的, 比如:

![img](http://oe3zwqfm1.bkt.clouddn.com/F77D9F6D91CB154531B40E2482861294.jpg)

#### Implementing a Test Service

我们需要建立一个测试服务提供这种 split input 情况

```js
"use strict";
const
  net = require('net'),
  server = net.createServer(function (connection) {
    console.log('Subscriber connected');
    // send the first chunk immediately
    connection.write(
      '{"type":"changed","file":"targ'
    );
    // after a one second delay, send the other chunk
    let timer = setTimeout(function () {
      connection.write('et.txt","timestamp":1358175758495}' + "\n");
      connection.end();
    }, 1000);
    // clear timer when the connection ends
    connection.on('end', function () {
      clearTimeout(timer);
      console.log('Subscriber disconnected');
    });
  });
server.listen(5432, function () {
  console.log('Test server listening for subscribers...');
});
```

### Extending Core Classes in Custom Modules

上面的错误情况, 是因为 client 没有缓存他的输入.

所以 client 应该有两个任务, 一个是缓存来的数据, 一个是处理到达的消息.

#### Extending EventEmitter

##### Inheritance in Node

```js
const
  events = require('events'),
  util = require('util'),
  // client constructor
  LDJClient = function (stream) {
    events.EventEmitter.call(this);
  };
util.inherits(LDJClient, events.EventEmitter);
```

LDJClient 是一个构造函数, 可以调用 `new LDJClient(stream)` 得到一个实例. `stream` 参数是一个对象, 可以 emit data事件, 比如一个 `Socket` 连接.

`events.EventEmitter.call(this);` 相当于 `super();`

#### Buffering Data Events

缓存数据

```js
LDJClient = function (stream) {
  events.EventEmitter.call(this);
  let
    self = this,
    buffer = '';
  stream.on('data', function (data) {
    buffer += data;
    let boundary = buffer.indexOf('\n');
    while (boundary !== -1) {
      let input = buffer.substr(0, boundary);
      buffer = buffer.substr(boundary + 1);
      self.emit('message', JSON.parse(input));
      boundary = buffer.indexOf('\n');
    }
  });
};
```

#### Exporting Functionality in a Module

```js
"use strict";
const
  events = require('events'),
  util = require('util'),
  // client constructor
  LDJClient = function (stream) {
    events.EventEmitter.call(this);
    let
      self = this,
      buffer = '';
    stream.on('data', function (data) {
      buffer += data;
      let boundary = buffer.indexOf('\n');
      while (boundary !== -1) {
        let input = buffer.substr(0, boundary);
        buffer = buffer.substr(boundary + 1);
        self.emit('message', JSON.parse(input));
        boundary = buffer.indexOf('\n');
      }
    });
  };
util.inherits(LDJClient, events.EventEmitter);

// expose module methods
exports.LDJClient = LDJClient;
exports.connect = function (stream) {
  return new LDJClient(stream);
};
```

`exports` 是两个模块之间的桥梁.

在 `exports` 上的属性可以再其它模块中使用.

在其它模块中可以这样调用:
```js
const
  ldj = require('./ldj.js'),
  client = ldj.connect(networkStream);
client.on('message', function (message) {
  // take action for this message
});
```

#### Importing a Custom Node Module

```js
"use strict";
const
  net = require('net'),
  ldj = require('./ldj.js'),

  netClient = net.connect({
    port: 5432
  }),
  ldjClient = ldj.connect(netClient);

ldjClient.on('message', function (message) {
  if (message.type === 'watching') {
    console.log("Now watching: " + message.file);
  } else if (message.type === 'changed') {
    console.log(
      "File '" + message.file + "' changed at " + new Date(message.timestamp)
    );
  } else {
    throw Error("Unrecognized message type: " + message.type);
  }
});
```

### Wrapping Up

目标是让代码 more testable, more robust, and more modular

在之前的server代码中, 每来一个connection, 就新建了一个watcher, 如果来了很多个connection, 就会新建很多个watcher. 如何从请求的连接中将watcher解耦出来?

自己写了一个sample:
```js
const fs = require('fs');
const net = require('net');
const resolve = require('path').resolve;
const filename = resolve(__dirname, process.argv[2]);

function Wather(fileName) {
  this.fileName = fileName;
  this.connectionArray = [];
  this.notify = fs.watchFile(fileName, { interval: 500 }, function () {
    this.connectionArray.map(function (item, index) {
      item.write(JSON.stringify({
        type: 'changed',
        file: fileName,
        timestamp: Date.now()
      }) + '\n');  
    }, this);
  }.bind(this));
}
Wather.prototype.addConnection = function (connection) {
  this.connectionArray.push(connection);
  connection.write(JSON.stringify({
    type: 'watching',
    file: this.fileName
  }) + '\n');
};
Wather.prototype.removeConnection = function (connection) {
  const index = this.connectionArray.indexOf(connection);
  this.connectionArray.splice(index, 1);
};

const watcher = new Wather(filename);

const server = net.createServer(function (connection) {
  console.log('Subscriber connected.');
  watcher.addConnection(connection);
  connection.on('close', function () {
    console.log('Subscriber disconnected.');
    watcher.removeConnection(connection);
  });
});
if (!filename) {
  throw Error('No target filename was specified.');
}
server.listen(5432, function () {
  console.log('Listening for subscribers...');
});
process.on('SIGINT', () => {
  server.close();
  process.exit();
});
```

## Robust Messaging Services

在这章中我们会使用第三方库, npm包 Zero-M-Q, 

### Advantages of ØMQ

### Importing External Modules with npm

模块可以通过纯就JavaScript编写, 也可以使用JavaScript联合其他语言, 依附语言通过对象动态联系起来.

### Message-Publishing and -Subscribing

#### Publishing Messages over TCP

使用 zeromq 编写一个 sub
```js
'use strict';
const
  fs = require('fs'),
  zmq = require('zeromq'),
  // create publisher endpoint
  publisher = zmq.socket('pub'),
  resolve = require('path').resolve,
  filename = resolve(__dirname, process.argv[2]);

fs.watch(filename, function () {
  // send message to any subscribers
  publisher.send(JSON.stringify({
    type: 'changed',
    file: filename,
    timestamp: Date.now()
  }));
});
// listen on TCP port 5432
publisher.bind('tcp://*:5432', function (err) {
  console.log('Listening for zmq subscribers...');
});
```

#### Subscribing to a Publisher
```js
"use strict";
const
  zmq = require('zeromq'),
  // create subscriber endpoint
  subscriber = zmq.socket('sub'); // subscribe to all messages
subscriber.subscribe("");
// handle messages from publisher
subscriber.on("message", function (data) {
  let
    message = JSON.parse(data),
    date = new Date(message.timestamp);
  console.log("File '" + message.file + "' changed at " + date);
});
// connect to publisher
subscriber.connect("tcp://localhost:5432");
```

#### Automatically Reconnecting Endpoints

一个终端跑 pub.js, 另一个终端跑 sub.js, 现象和之前一样.

这里有一个神奇的现象, 我们kill掉 pub.js 以后, sub 没有发生任何变化, 

zeromq 中, 不关心哪一端先启动.

在我们的代码里, publisher 绑定一个TCP socket, subscriber 连接, 不过 zeromq 不限制如此, 你也可以反转他们, subscriber 绑定一个 socket 而 publisher 进行连接.

在实际的应用中, 稳定的部分通常作为绑定的一方, 而短暂的部分通常作为进行连接的一方.

### Responding to Requests

REQ/REP(request/reply) 模式.

在 zeromq 中, REQ/REP 模式是进行锁步式的交流, request 来了, 会放进队列里, 一次只会 reply 一个 request, 也就是说, 应用同一时间只会处理一个 request.

#### Implementing a Responder

```js
'use strict';
const
  fs = require('fs'),
  zmq = require('zeromq'),
  // socket to reply to client requests
  responder = zmq.socket('rep');
// handle incoming requests
responder.on('message', function (data) {

  // parse incoming message
  let request = JSON.parse(data);
  console.log('Received request to get: ' + request.path);

  // read file and reply with content
  fs.readFile(request.path, function (err, content) {
    console.log('Sending response content');
    responder.send(JSON.stringify({
      content: content.toString(),
      timestamp: Date.now(),
      pid: process.pid
    }));
  });

});

// listen on TCP port 5433
responder.bind('tcp://127.0.0.1:5433', function (err) {
  console.log('Listening for zmq requesters...');
});

// close the responder when the Node process ends
process.on('SIGINT', function () {
  console.log('Shutting down...');
  responder.close();
});
```

#### Issuing Requests
```js
"use strict";
const
  zmq = require('zeromq'),
  resolve = require('path').resolve,
  filename = resolve(__dirname, process.argv[2]),
  // create request endpoint
  requester = zmq.socket('req');
// handle replies from responder
requester.on("message", function(data) {
  let response = JSON.parse(data);
  console.log("Received response:", response);
});
requester.connect("tcp://localhost:5433");
// send request for content
console.log('Sending request for ' + filename);
requester.send(JSON.stringify({
  path: filename
}));
```

以上, 监听 message 事件, 与rep socket建立tcp连接, 最后调用 `requester.send()`, 

#### Trading Synchronicity for Scale

