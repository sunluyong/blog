# Duplex

双工流就是同时实现了 Readable 和 Writable 的流，即可以作为上游生产数据，又可以作为下游消费数据，这样可以处于数据流动管道的中间部分，即

```javascript
 rs.pipe(rws1).pipe(rws2).pipe(rws3).pipe(ws);
```

在 NodeJS 中双工流常用的有两种

1. Duplex
2. Transform

## Duplex

### 实现 Duplex

和 Readable、Writable 实现方法类似，实现 Duplex 流非常简单，但 Duplex 同时实现了 Readable 和 Writable， NodeJS 不支持多继承，所以我们需要继承 Duplex 类

1. 继承 Duplex 类
2. 实现 _read() 方法
3. 实现 _write() 方法

相信大家对 *read()、*write() 方法的实现不会陌生，因为和 Readable、Writable 完全一样。

```javascript
const Duplex = require('stream').Duplex;

const myDuplex = new Duplex({
  read(size) {
    // ...
  },
  write(chunk, encoding, callback) {
    // ...
  }
});
```

### 构造函数参数

Duplex 实例内同时包含可读流和可写流，在实例化 Duplex 类的时候可以传递几个参数

- readableObjectMode <Boolean> : 可读流是否设置为 ObjectMode，默认 false
- writableObjectMode  <Boolean>: 可写流是否设置为 ObjectMode，默认 false
- allowHalfOpen <Boolean> : 默认 true， 设置成 false 的话，当写入端结束的时，流会自动的结束读取端，反之亦然。

### 小例子

了解了 Readable 和 Writable 之后看 Duplex 非常简单，直接用一个官网的例子

```javascript
 const Duplex = require('stream').Duplex;
const kSource = Symbol('source');

class MyDuplex extends Duplex {
  constructor(source, options) {
    super(options);
    this[kSource] = source;
  }

  _write(chunk, encoding, callback) {
    // The underlying source only deals with strings
    if (Buffer.isBuffer(chunk))
      chunk = chunk.toString();
    this[kSource].writeSomeData(chunk);
    callback();
  }

  _read(size) {
    this[kSource].fetchSomeData(size, (data, encoding) => {
      this.push(Buffer.from(data, encoding));
    });
  }
}
```

当然这是不能执行的伪代码，但是 Duplex 的作用可见一斑，进可以生产数据，又可以消费数据，所以才可以处于数据流动管道的中间环节，常见的 Duplex 流有

- Tcp Scoket
- Zlib
- Crypto

## Transform

Transform 同样是双工流，看起来和 Duplex 重复了，但两者有一个重要的区别：Duplex 虽然同事具备可读流和可写流，但两者是相对独立的；Transform 的可读流的数据会经过一定的处理过程自动进入可写流。

虽然会从可读流进入可写流，但并不意味这两者的数据量相同，上面说的一定的处理逻辑会决定如果 tranform 可读流，然后放入可写流，transform 原义即为转变，很贴切的描述了 Transform 流作用。

我们最常见的压缩、解压缩用的 zlib 即为 Transform 流，压缩、解压前后的数据量明显不同，儿流的作用就是输入一个 zip 包，输入一个解压文件或反过来。我们平时用的大部分双工流都是 Transform。

### 实现 Tranform

Tranform 类内部继承了 Duplex 并实现了 writable.write() 和 readable._read() 方法，我们想自定义一个 Transform 流，只需要

1. 继承 Transform 类
2. 实现 _transform() 方法
3. 实现 _flush() 方法（可以不实现）

_transform(chunk, encoding, callback) 方法用来接收数据，并产生输出，参数我们已经很熟悉了，和 Writable 一样， chunk 默认是 Buffer，除非 decodeStrings 被设置为 false。

在 _transform() 方法内部可以调用 this.push(data) 生产数据，交给可写流，也可以不调用，意味着输入不会产生输出。

当数据处理完了必须调用 callback(err, data)  ，第一个参数用于传递错误信息，第二个参数可以省略，如果被传入了，效果和 this.push(data) 一样

```javascript
 transform.prototype._transform = function (data, encoding, callback) {
  this.push(data);
  callback();
};

transform.prototype._transform = function (data, encoding, callback) {
  callback(null, data);
};
```

有些时候，transform 操作可能需要在流的最后多写入可写流一些数据。例如， Zlib流会存储一些内部状态，以便优化压缩输出。在这种情况下，可以使用_flush()方法，它会在所有写入数据被消费、触发 'end'之前被调用。

## Transform 事件

Transform 流有两个常用的事件

1. 来自 Writable 的 finish
2. 来自 Readable 的 end

当调用 transform.end() 并且数据被 _transform() 处理完后会触发 finish，调用_flush后，所有的数据输出完毕，触发end事件。

## 对比

了解了 Readable 和 Writable 之后，理解双工流十分自然，但两者的区别会让一些初学者困惑，简单的区分：Duplex 的可读流和可写流之间并没有直接关系，Transform 中可读流的数据会经过处理后自动放入可写流中。

看两个简单的例子就能直观了解到 Duplex 和 Transform 的区别

#### TCP socket

net 模块可以用来创建 socket，socket 在 NodeJS 中是一个典型的 Duplex，看一个 TCP 客户端的例子

```javascript
var net = require('net');

//创建客户端
var client = net.connect({port: 1234}, function() {
    console.log('已连接到服务器');
    client.write('Hi!');
});

//data事件监听。收到数据后，断开连接
client.on('data', function(data) {
    console.log(data.toString());
    client.end();
});

//end事件监听，断开连接时会被触发
client.on('end', function() {
    console.log('已与服务器断开连接');
});
```

可以看到 client 就是一个 Duplex，可写流用于向服务器发送消息，可读流用于接受服务器消息，两个流内的数据并没有直接的关系。

#### gulp

gulp 非常擅长处理代码本地构建流程，看一段官网的示例代码

```javascript
gulp.src('client/templates/*.jade')
  .pipe(jade())
  .pipe(minify())
  .pipe(gulp.dest('build/minified_templates'));
```

其中 jada() 和 minify() 就是典型的 Transform，处理流程大概是

```javascript
.jade 模板文件 -> jada() -> html 文件 -> minify -> 压缩后的 html
```

可以看出来，jade() 和 minify() 都是对输入数据做了些特殊处理，然后交给了输出数据。

这样简单的对比就能看出 Duplex 和 Transform 的区别，在平时实用的时候，当一个流同事面向生产者和消费者服务的时候我们会选择 Duplex，当只是对数据做一些转换工作的时候我们便会选择使用 Tranform。