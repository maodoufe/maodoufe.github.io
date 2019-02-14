---
layout: post
title: Node Stream
categories: [node.js, javascript]
tags: [Stream, Node.js]
fullview: true
comments: true
---

作者：qianniuer@[毛豆前端](https://maodoufe.github.io/)

# 背景
从早先的unix开始，stream便开始进入了人们的视野，在过去的几十年的时间里，它被证明是一种可依赖的编程方式，它可以将一个大型的系统拆成一些很小的部分，并且让这些部分之间完美地进行合作。在unix中，我们可以使用|符号来实现流。在node中，node内置的stream模块已经被多个核心模块使用，同时也可以被用户自定义的模块使用。和unix类似，node中的流模块的基本操作符叫做.pipe()，同时你也可以使用一个后压机制来应对那些对数据消耗较慢的对象。
# 为什么应该使用流?

在node中，I/O都是异步的，所以在和硬盘以及网络的交互过程中会涉及到传递回调函数的过程。你之前可能会写出这样的代码
```
let http = require('http');
let fs = require('fs');

let server = http.createServer(function (req, res) {
	fs.readFile(__dirname + 'data.txt', function (err, data) {
	    res.end(data);
	});
});
server.listen(8000);
```
上面的这段代码并没有什么问题，但是在每次请求时，我们都会把整个data.txt文件读入到内存中，然后再把结果返回给客户端。想想看，如果data.txt文件非常大，在响应大量用户的并发请求时，程序可能会消耗大量的内存，这样很可能会造成用户连接缓慢的问题。

其次，上面的代码可能会造成很不好的用户体验，因为用户在接收到任何的内容之前首先需要等待程序将文件内容完全读入到内存中。

所幸的是，(req,res)参数都是流对象，这意味着我们可以使用一种更好的方法来实现上面的需求：
```
let http = require('http');
let fs = require('fs');

let server = http.createServer(function (req, res) {
	let stream = fs.createReadStream(__dirname + '/data.txt');
	stream.pipe(res);
});
server.listen(8000);
```
在这里，.pipe()方法会自动帮助我们监听data和end事件。上面的这段代码不仅简洁，而且data.txt文件中每一小段数据都将源源不断的发送到客户端。

除此之外，使用.pipe()方法还有别的好处，比如说它可以自动控制后端压力，以便在客户端连接缓慢的时候node可以将尽可能少的缓存放到内存中。

想要将数据进行压缩？我们可以使用相应的流模块完成这项工作!
```
let http = require('http');
let fs = require('fs');
let oppressor = require('oppressor');

let server = http.createServer(function (req, res) {
	let stream = fs.createReadStream(__dirname + '/data.txt');
	stream.pipe(oppressor(req)).pipe(res);
});
server.listen(8000);
```
通过上面的代码，我们成功的将发送到浏览器端的数据进行了gzip压缩。我们只是使用了一个oppressor模块来处理这件事情。

一旦你学会使用流api，你可以将这些流模块像搭乐高积木或者像连接水管一样拼凑起来，从此以后你可能再也不会去使用那些没有流API的模块获取和推送数据了。
## 01 流的概念
* 流是一组有序的，有起点和终点的字节数据传输手段
* 能够更好的控制读取和写入的速度

## 02 开始使用
1. 可读流createReadStream
* 流的两种模式： 暂停模式
```
// 可读流
let fs = require('fs');
// test.js   123456789
let rs = fs.createReadStream('./test.js', {
 	flags: 'r', // 读取方式
 	encoding: null, // 编码 默认 buffer
 	autoClose: true,
 	mode: 0666, // 权限 默认 0666
 	start: 0, // 开始读取位置
 	end: 3,  // 结束位置 包后
 	highWaterMark: 2,  // 最高水位线
})
// 默认什么都不干 结果默认是不会读取的
```
* 流的两种模式： 流动模式
```
rs.on('data', function (data) {
   console.log(data) // <Buffer 31 32> <Buffer 33 34> 分两次输出
})
rs.on('end', function() {
   console.log('读取完毕')
})
```
* 暂停和恢复 可控制读取数据的速度
```
let arr = []
rs.on('data', function (data) {
   rs.pause() // 暂停 暂停触发data事件
   arr.push(data);
})
rs.on('end', function() {
   console.log(Buffer.concat(arr).toString()); // 1234
})
setTimeout(() => {
   rs.resume(); // 恢复流
}, 1000)
```
```
// 错误监听
rs.on('error', function (err) {
   console.log(err)
})
```
2. 可写流
* 写 （第一次会真的往文件里写）后面会写到缓存中
```
// 可写流
let fs = require('fs');
let ws = fs.createWriteStream('2.txt', {
        flags: 'w',
 	encoding: 'utf8',
 	autoClose: true,
 	start: 5,
 	highWaterMark: 3
});
```
* 上面运行结果与highWaterMark有关， flag 代表 是否小于最高水位线
```
let flag = ws.write('1')
console.log(flag); // true
flag = ws.write('1')
console.log(flag); // true
flag = ws.write('1')
console.log(flag); // false
```
**highWaterMark**
 >highWaterMark只是一个标识，一般配合着读取来用，例如：
预计用上述highWaterMark内存容量，假如有个文件 1g大小，每次读取64k，读取后超出最高水位线应该暂停一下 rs.pause()，若不暂停，会导致缓存过大

```
// 抽干, 即缓存中文件已被全部写入文件中（前提是，当前写入内容 >= highWaterMark）
	ws.on('drain', function(){
 		console.log('已抽干')
	})
	ws.end('结束') // 会将缓存区的内容清空后再关闭文件，且也会写入此方法中的内容
	ws.write('ok') // ❌ 不能在 end 后继续写入
```

## 03 流的原理／手把手教你写流的简版源码
1. 可读流
流的读取是通过多个事件监听来实现的，可读流的常用事件：
```
let fs = require('fs');
let rs = fs.createReadStream('./2.txt', {
 	highWaterMark: 3,
 	autoClose: true,
 	flags: 'r',
 	start: 0,
 	end: 5,
 	// encoding: 'utf8', // 默认 buffer
})
rs.on('open', function() {
    console.log('open')
});
rs.on('error', function (err) {
    console.log(err);
})
rs.on('data', function(data) {
    console.log(data);
    rs.pause()
})
rs.on('end', function() {
    console.log('end')
})
rs.on('close', function(){
    console.log('close')
})
setInterval(() =>{
 	rs.resume();
}, 1000)
```
2. 可读流的实现
```
let EventEmitter = require('events');
let fs = require('fs');
class ReadStream extends EventEmitter {
    constructor(path, options) {
		super();
		this.path = path;
		this.autoClose = options.autoClose || true;
		this.flags = options.flags || 'r';
		this.encoding = options.encoding || null;
		this.start = options.start || 0;
		this.end = options.end || null;
		this.highWaterMark = options.highWaterMark || 64 * 1024;
		// 应该有一个读取文件的位置 可变的（可变的位置）
		this.pos = this.start;
		// 控制当前是否是流动模式
		this.flowing = null;
		// 构建读取到的内容的buffer
		this.buffer = Buffer.alloc(this.highWaterMark);
		// 但创建可读流 要将文件打开
		this.open();
		this.on('newListener', (type) => {
			if (type === 'data') { 
			// 如果用户监听了data事件，就开始读取
			this.flowing = true;
			this.read(); // 开始读取文件
			}
		})
    }
    read() {
		// 这时候文件还没有打开，等文件打开后再去读取
		if (typeof this.fd !== 'number') {
			// 等待文件打开，再次调用read方法
			return this.once('open', () => this.read())
		}
			// 开始读取
		let howMuchToRead = this.end ? Math.min(this.highWaterMark, this.end - this.pos + 1) : this.highWaterMark;
		// 文件描述符 读到哪个buffer里  读取到buffer的哪个位置 往buffer里读取几个 读取的位置
		// 想读三个 但是文件只有2个
		// todo 外部调用resume方法的时候 需判断如果end了 就不要读了
		fs.read(this.fd, this.buffer, 0, howMuchToRead, this.pos, (err, bytesRead) => {
			if (err) {
				this.emit('error')
				return
			}
			if (bytesRead > 0) { // 读到内容了
				this.pos += bytesRead;
				// 保留有用的
				let r = this.buffer.slice(0, bytesRead);
				r = this.encoding ? r.toString(this.encoding) : r;
				this.emit('data', r);
				if (this.flowing) {
					this.read();
				}
			} else {
				this.emit('end');
				this.destroy();
			}
		});
    }
    destroy() {
		if (typeof this.fd === 'number') {
			fs.close(this.fd, () => {
				this.emit('close');
			});
			return
		}
		this.emit('close');
    }
    open() { // 打开文件的逻辑
		fs.open(this.path, this.flags, (err, fd) => {
			if (err) {
				this.emit('error', err);
				if (this.autoClose) {
					this.destroy() // 销毁  关闭文件 （触发close事件）
				}
				return
			}
			this.fd = fd;
			this.emit('open'); // 触发文件开启事件
		})
    }
    pause() {
        this.flowing = false;
    }
    resume() {
        this.flowing = true;
        this.read();
    }
}
module.exports = ReadStream;
```
3. 可写流
```
let fs = require('fs');
//  我想写入时只占用三个字节的内存
let ws = fs.createWriteStream('./1.txt', {
    flags: 'w',
    mode: 0o666,
    highWaterMark: 3,
    start: 0,
    autoClose: true,
    encoding: 'utf8',
})
let i = 0;
function write() {
    let flag = true;
    while(i < 9 && flag) {
        flag = ws.write(i++ + '')
    }
}
ws.on('drain', function () {
    console.log('写入成功')
    write()
})
```
4. 可写流的实现
```
let fs = require('fs');
let EventEmitter = require('events');
class WriteStream extends EventEmitter {
    constructor(path, options) {
        super();
        this.path = path;
        this.flags = options.flags || 'w';
        this.mode = options.mode || 0o666;
        this.highWaterMark = options.highWaterMark || 16 * 1024;
        this.start = options.start || 0;
        this.autoClose = options.autoClose || true;
        this.encoding = options.encoding || 'utf8';
        // 是否需要触发drain事件
        this.needDrain = false;
        // 是否正在写入
        this.writing = false;
        // 缓存 正在写入就放到缓存中
        this.buffer = [];
        // 算一个当前缓存的个数
        this.len = 0;
        // 写入 的时候也有位置关系
        this.pos = this.start;
        this.open();
    }
    open() {
        fs.open(this.path, this.flags, this.mode, (err, fd) => {
            if (err) {
                this.emit('error');
                this.destroy();
                return
            }
            this.fd = fd;
            this.emit('open');
        })
    }
    destroy() {
        if (typeof this.fd === 'number') {
            fs.close(this.fd, () => {
                this.emit('close');
            });
            return
        }
        this.emit('close');
    }
    write(chunk, encoding = this.encoding, callback) {
        chunk = Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk);
        this.len += chunk.length;
        this.needDrain = this.highWaterMark <= this.len;
        if (this.writing) {
            this.buffer.push({chunk, encoding, callback})
        } else {
            // 当文件写入沟 清空缓存区的内容
            this.writing = true; // 走缓存
            this._write(chunk, encoding, () => this.clearBuffer())
        }
        return !this.needDrain;
    }
    _write(chunk, encoding, callback) {
        if (typeof this.fd !== 'number') {
            return this.once('open', () => this.write(chunk, encoding, callback));
        }
        // fd是文件描述符 chunk是数据  0 是写入的位置  写入的长度， this.pos 偏移量
        fs.write(this.fd, chunk, 0, chunk.length, this.pos, (err, bytesWritten) => {
            this.pos += bytesWritten;
            this.len -= bytesWritten; // 写入的长度会减少
            callback();
        })
    }
    clearBuffer() {
        let buf = this.buffer.shift();
        if(buf) {
            this._write(buf.chunk, buf.encoding, () => this.clearBuffer());
        } else {
            this.writing = false;
            this.needDrain = false;
            this.emit('drain')
        }
    }
}
module.exports = WriteStream;
```