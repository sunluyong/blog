# 什么是 Stream

对于大部分有后端经验的的同学来说 Stream 对象是个再合理而常见的对象，但对于前端同学 Stream 并不是那么理所当然，github 上甚至有一篇 9000 多 Star 的文章介绍到底什么是 Stream —— [stream-handbook](https://link.zhihu.com/?target=https%3A//github.com/substack/stream-handbook)。为了更好的理解 Stream，在这篇文章的基础上简单总结概括一下。



## 什么是 Stream

在 Unix 系统中流就是一个很常见也很重要的概念，从术语上讲流是对输入输出设备的抽象。

```shell
ls | grep *.js
```

类似这样的代码我们在写脚本的时候经常可以遇到，使用 **`|` **连接两条命令，把前一个命令的结果作为后一个命令的参数传入，这样数据像是水流在管道中传递，每个命令类似一个处理器，对数据做一些加工，因此 **|** 被称为 “**管道符号**”。



## NodeJS 中 Stream 的几种类型

从程序角度而言流是**有方向的数据**，按照流动方向可以分为三种流

1. 设备流向程序：readable
2. 程序流向设备：writable
3. 双向：duplex、transform

NodeJS 关于流的操作被封装到了 [Stream](https://link.zhihu.com/?target=http%3A//nodejs.org/docs/latest/api/stream.html) 模块，这个模块也被多个核心模块所引用。按照 Unix 的哲学：一切皆文件，在 NodeJS 中对文件的处理多数使用流来完成

1. 普通文件
2. 设备文件（stdin、stdout）
3. 网络文件（http、net）

有一个很容易忽略的知识点：在 NodeJS 中所有的 Stream 都是 [EventEmitter](https://link.zhihu.com/?target=https%3A//nodejs.org/dist/latest/docs/api/events.html%23events_class_eventemitter) 的实例。



## 小例子

我们写程序忽然需要读取某个配置文件 **config.json**，这时候简单分析一下

- 数据：config.json 的内容
- 方向：设备（物理磁盘文件） -> NodeJS 程序

我们应该使用 readable 流来做此事

```javascript
const fs = require('fs');
const FILEPATH = '...';

const rs = fs.createReadStream(FILEPATH);
```

通过 fs 模块提供的 `createReadStream()` 方法我们轻松的创建了一个可读的流，这时候  **config.json** 的内容从设备流向程序。我们并没有直接使用 Stream 模块，因为 fs 内部已经引用了 Stream 模块，并做了封装。

有了数据后我们需要处理，比如需要写到某个路径 DEST ，这时候我们遍需要一个 writable 的流，让数据从程序流向设备。

```javascript
const ws = fs.createWriteStream(DEST);
```

两种流都有了，也就是两个数据加工器，那么我们如何通过类似 Unix 的管道符号 `|` 来链接流呢？在 NodeJS 中管道符号就是 `pipe()` 方法。

```javascript
const fs = require('fs');
const FILEPATH = '...';

const rs = fs.createReadStream(FILEPATH);
const ws = fs.createWriteStream(DEST);

rs.pipe(ws);
```

这样我们利用流实现了简单的文件复制功能，关于 pipe() 方法的实现原理后面会提到，但有个值得注意地方：数据必须是从上游 pipe 到下游，也就是从一个 readable 流 pipe 到 writable 流。



## 加工一下数据

上面提到了 readable 和 writable 的流，我们称之为加工器，其实并不太恰当，因为我们并没有加工什么，只是读取数据，然后存储数据。

如果有个需求，把本地一个 *package.json* 文件中的所有字母都改为小写，并保存到同目录下的 package-lower.json 文件下。

这时候我们就需要用到双向的流了，假定我们有一个专门处理字符转小写的流 lower，那么代码写出来大概是这样的

```javascript
const fs = require('fs');

const rs = fs.createReadStream('./package.json');
const ws = fs.createWriteStream('./package-lower.json');

rs.pipe(lower).pipe(ws);
```

这时候我们可以看出为什么称 pipe() 连接的流为加工器了，根据上面说的，必须从一个 readable 流 pipe 到 writable 流：

- rs -> lower：lower 在下游，所以 lower 需要是个 writable 流
- lower -> ws：相对而言，lower 又在上游，所以 lower 需要是个 readable 流

有点推理的赶脚呢，能够满足我们需求的 lower 必须是双向的流，具体使用 duplex 还是 transform 后面我们会提到。

当然如果我们还有额外一些处理动作，比如字母还需要转成 ASCII 码，假定有一个流 ascii 那么我们代码可能是

```javascript
rs.pipe(lower).pipe(acsii).pipe(ws);
```

同样 ascii 也必须是双向的流。这样处理的逻辑是非常清晰的，那么除了代码清晰，使用流还有什么好处呢？



## 为什么应该使用 Stream

有个用户需要在线看视频的场景，假定我们通过 HTTP 请求返回给用户电影内容，那么代码可能写成这样

```javascript
const http = require('http');
const fs = require('fs');

http.createServer((req, res) => {
   fs.readFile(moviePath, (err, data) => {
      res.end(data);
   });
}).listen(8080);
```

这样的代码又两个明显的问题

1. 电影文件需要读完之后才能返回给客户，等待时间超长
2. 电影文件需要一次放入内存中，相似动作多了，内存吃不消

用流可以讲电影文件一点点的放入内存中，然后一点点的返回给客户（利用了 HTTP 协议的 Transfer-Encoding: chunked 分段传输特性），用户体验得到优化，同时对内存的开销明显下降

```javascript
const http = require('http');

const fs = require('fs');

http.createServer((req, res) => {

   fs.createReadStream(moviePath).pipe(res);

}).listen(8080);
```

除了上述好处，代码优雅了很多，拓展也比较简单。比如需要对视频内容压缩，我们可以引入一个专门做此事的流，这个流不用关心其它部分做了什么，只要是接入管道中就可以了

```javascript
const http = require('http');

const fs = require('fs');

const oppressor = require(oppressor);

http.createServer((req, res) => {

   fs.createReadStream(moviePath)

      .pipe(oppressor)

      .pipe(res);

}).listen(8080);
```

可以看出来，使用流后，我们的代码逻辑变得相对独立，可维护性也会有一定的改善，关于几种流的具体使用方式且听下回分解。