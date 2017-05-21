# Readable

## 什么是可读流

可读流是**生产数据**用来供程序消费的流。我们常见的数据生产方式有读取磁盘文件、读取网络请求内容等，看一下前面介绍什么是流用的例子：

```javascript
const rs = fs.createReadStream(filePath);
```

rs 就是一个可读流，其生产数据的方式是读取磁盘的文件，我们常见的控制台 process.stdin 也是一个可读流：

```javascript
process.stdin.pipe(process.stdout);
```

通过简单的一句话可以把控制台的输入打印出来，process.stdin 生产数据的方式是读取用户在控制台的输入。

回头再看一下我们对可读流的定义：可读流是生产数据用来供程序消费的流。

## 自定义可读流

除了系统提供给我们的 `fs.CreateReadStream` 我们还经常使用 gulp 或者 vinyl-fs 提供的 src 方法

```javascript
gulp.src(['*.js', 'dist/**/*.scss'])
```

如果我们想自己以某种特定的方式生产数据，交给程序消费，那么改如何开始呢？

简单两步即可

1. 继承 sream 模块的 **Readable** 类
2. 重写 **_read** 方法，调用 **this.push** 将生产的数据放入待读取队列

Readable 类已经把可读流要做的大部分工作完成，我们只需要继承它，然后把生产数据的方式写在 _read 方法里就可以实现一个自定义的可读流。

如果我们想实现一个每 100 毫秒生产一个随机数的流（没什么用处）

```javascript
const Readable = require('stream').Readable;

class RandomNumberStream extends Readable {
    constructor(max) {
        super()
    }

    _read() {
        const ctx = this;
        setTimeout(() => {
            const randomNumber = parseInt(Math.random() * 10000);

            // 只能 push 字符串或 Buffer，为了方便显示打一个回车
            ctx.push(`${randomNumber}\n`);
        }, 100);
    }
}

module.exports = RandomNumberStream;
```

类继承部分代码很简单，主要看一下 _read 方法的实现，有几个值得注意的地方

1. Readable 类中默认有 _read 方法的实现，不过什么都没有做，我们做的是覆盖重写
2. _read 方法有一个参数 size，用来向 read 方法指定应该读取多少数据返回，不过只是一个参考数据，很多实现忽略此参数，我们这里也忽略了，后面会详细提到
3. 通过 this.push 向缓冲区推送数据，缓冲区概念后面会提到，暂时理解为挤到了水管中可消费了
4. push 的内容只能是字符串或者 Buffer，不能是数字
5. push 方法有第二个参数 encoding，用于第一个参数是字符串时指定 encoding

执行一下看看效果

```javascript
const RandomNumberStream = require('./RandomNumberStream');

const rns = new RandomNumberStream();

rns.pipe(process.stdout);
```

这样可以看到数字源源不断的显示到了控制台上，我们实现了一个产生随机数的可读流，还有几个小问题待解决

## 如何停下来

我们每隔 100 毫秒向缓冲区推送一个数字，那么就像读取一个本地文件总有读完的时候，如何停下来标识数据读取完毕？

向缓冲区 push 一个 null 就可以。我们修改一下代码，允许消费者定义需要多少个随机数字：

```javascript
const Readable = require('stream').Readable;

class RandomNumberStream extends Readable {
    constructor(max) {
        super()
        this.max = max;
    }

    _read() {
        const ctx = this;

        setTimeout(() => {
            if (ctx.max) {
                const randomNumber = parseInt(Math.random() * 10000);

                // 只能 push 字符串或 Buffer，为了方便显示打一个回车
                ctx.push(`${randomNumber}\n`);
                ctx.max -= 1;
            } else {
                ctx.push(null);
            }
        }, 100);
    }
}

module.exports = RandomNumberStream;
```

我们使用了一个 max 的标识，允许消费者指定需要的字符数，在实例化的时候指定即可

```javascript
const RandomNumberStream = require('./RandomNumberStream');

const rns = new RandomNumberStream(5);

rns.pipe(process.stdout);
```

这样可以看到控制台只打印了 5 个字符

## 为什么是 setTimeout 而不是 setInterval

细心的同学可能注意到，我们每隔 100 毫秒生产一个随机数并不是调用的 setInterval，而是使用的 setTimeout，为什么仅仅是延时了一下并没有重复生产，结果却是正确的呢？

这就需要了解流的两种工作方式

1. 流动模式：数据由底层系统读出，并尽可能快地提供给应用程序
2. 暂停模式：必须显示地调用 read() 方法来读取若干数据块

流在默认状态下是处于暂停模式的，也就是需要程序显式的调用 read() 方法，可我们的例子中并没有调用就可以得到数据，因为我们的流通过 pipe() 方法切换成了流动模式，这样我们的 _read() 方法会自动被反复调用，直到数据读取完毕，所以我们每次 _read() 方法里面只需要读取一次数据即可。

## 流动模式和暂停模式切换

流从默认的暂停模式切换到流动模式可以使用以下几种方式：

1. 通过添加 data 事件监听器来启动数据监听
2. 调用 resume() 方法启动数据流
3. 调用 pipe() 方法将数据转接到另一个 可写流

从流动模式切换为暂停模式又两种方法：

1. 在流没有 pipe() 时，调用 pause() 方法可以将流暂停
2. pipe() 时，需要移除所有 data 事件的监听，再调用 unpipe() 方法

## data 事件

使用了 pipe() 方法后数据就从可读流进入了可写流，但对我们好像是个黑盒，数据究竟是怎么流向的呢？我们看到切换流动模式和暂停模式的时候有两个重要的名词

1. 流动模式对应的 data 事件
2. 暂停模式对应的 read() 方法

这两个机制是我们能够驱动数据流动的原因，先来看一下流动模式 data 事件，一旦我们监听了可读流的 data 时、事件，流就进入了流动模式，我们可以改写一下上面调用流的代码

```javascript
const RandomNumberStream = require('./RandomNumberStream');

const rns = new RandomNumberStream(5);

rns.on('data', chunk => {
  console.log(chunk);
});
```

这样我们可以看到控制台打印出了类似下面的结果

```
<Buffer 39 35 37 0a>
<Buffer 31 30 35 37 0a>
<Buffer 38 35 31 30 0a>
<Buffer 33 30 35 35 0a>
<Buffer 34 36 34 32 0a>

```

当可读流生产出可供消费的数据后就会触发 data 事件，data 事件监听器绑定后，数据会被尽可能地传递。data 事件的监听器可以在第一个参数收到可读流传递过来的 Buffer 数据，这也就是我们打印的 chunk，如果想显示为数字，可以调用 Buffer 的 toString() 方法。

当数据处理完成后还会触发一个 **end** 事件，应为流的处理不是同步调用，所以如果我们希望完事后做一些事情就需要监听这个事件，我们在代码最后追加一句：

```javascript
rns.on('end', () => {
  console.log('done');
});
```

这样可以在数据接收完了显示 'done'

当然数据处理过程中出现了错误会触发 **error** 事件，我们同样可以监听，做异常处理：

```javascript
rns.on('error', (err) => {
  console.log(err);
});
```

## read(size) 

流在暂停模式下需要程序显式调用 read() 方法才能得到数据。read() 方法会从内部缓冲区中拉取并返回若干数据，当没有更多可用数据时，会返回null。

使用 read() 方法读取数据时，如果传入了 size 参数，那么它会返回指定字节的数据；当指定的size字节不可用时，则返回null。如果没有指定size参数，那么会返回内部缓冲区中的所有数据。

现在有一个矛盾了，在流动模式下流生产出了数据，然后触发 data 事件通知给程序，这样很方便。在暂停模式下需要程序去读取，那么就有一种可能是读取的时候还没生产好，如果我们才用轮询的方式未免效率有些低。

NodeJS 为我们提供了一个 **readable** 的事件，事件在可读流准备好数据的时候触发，也就是先监听这个事件，收到通知又数据了我们再去读取就好了：

```javascript
const rns = new RandomNumberStream(5);

rns.on('readable', () => {
  let chunk;
  while((chunk = rns.read()) !== null){
    console.log(chunk);
  }
});
```

这样我们同样可以读取到数据，值得注意的一点是并不是每次调用 read() 方法都可以返回数据，前面提到了如果可用的数据没有达到 size 那么返回 null，所以我们在程序中加了个判断。

## 数据会不会漏掉

开始使用流动模式的时候我经常会担心一个问题，上面代码中可读流在创建好的时候就生产数据了，那么会不会在我们绑定 readable 事件之前就生产了某些数据，触发了 readable 事件，我们还没有绑定，这样不是极端情况下会造成开头数据的丢失嘛

可事实并不会，按照 NodeJS event loop 我们创建流和调用事件监听在一个事件队列里面，儿生产数据由于涉及到异步操作，已经处于了下一个事件队列，我们监听事件再慢也会比数据生产块，数据不会丢失。

看到这里，大家其实对 data事件、readable事件触发时机， read() 方法每次读多少数据，什么时候返回 null 还有又一定的疑问，因为到现在为止我们接触到的仍然是一个黑盒，后面我们介绍了可写流后会在 back pressure 机制部分对这些内部细节结合源码详细讲解，且听下回分解吧。