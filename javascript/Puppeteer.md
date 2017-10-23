# Puppeteer: 更友好的 Headless Chrome Node API  

很早很早之前，前端就有了对 headless 浏览器的需求，最多的应用场景有两个

1. UI 自动化测试：摆脱手工浏览点击页面确认功能模式
2. 爬虫：解决页面内容异步加载等问题

也就有了很多杰出的实现，前端经常使用的莫过于 [PhantomJS](http://phantomjs.org/) 和 [selenium-webdriver](http://seleniumhq.github.io/selenium/docs/api/javascript/)，但两个库有一个共性——难用！环境安装复杂，API 调用不友好，1027 年 Chrome 团队连续放了两个大招 [Headless Chrome](https://chromium.googlesource.com/chromium/src/+/lkgr/headless/README.md) 和对应的 NodeJS API [Puppeteer](https://github.com/GoogleChrome/puppeteer)，直接让 PhantomJS 和 Selenium IDE for Firefox 作者悬宣布没必要继续维护其产品

## Puppeteer

如同其 [github](https://github.com/GoogleChrome/puppeteer) 项目介绍：Puppeteer 是一个通过  [DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) 控制 headless chrome 的 high-level Node 库，也可以通过设置使用 非 headless Chrome

我们手工可以在浏览器上做的事情 Puppeteer 都能胜任

1. 生成网页截图或者 PDF
2. 爬取大量异步渲染内容的网页，基本就是人肉爬虫
3. 模拟键盘输入、表单自动提交、UI 自动化测试

官方提供了一个 [playground](https://try-puppeteer.appspot.com/)，可以快速体验一下。关于其具体使用不在赘述，官网的 demo 足矣让完全不了解的同学入门

```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://example.com');
  await page.screenshot({path: 'example.png'});

  await browser.close();
})();
```

实现网页截图就这么简单，自己也实现了一个简单的[爬取百度图片的搜索结果的 demo](https://github.com/Samaritan89/headless-crawler/blob/master/src/mn.js)，代码不过 40 行，用过 selenium-webdriver 的同学看了会流泪，接下来介绍几个好玩的特性

## 哲学

虽然 Puppeteer API 足够简单，但如果是从 webdriver 流转过来的同学会很不适应，主要是在 webdirver 中我们操作网页更多的是从程序的视角，而在 Puppeteer 中网页浏览者的视角。举个简单的例子，我们希望对一个表单的 input 做输入

webdriver 流程

1. 通过选择器找到页面 input 元素
2. 给元素设置值

```javascript
    const input = await driver.findElement(By.id('kw'));
    await input.sendKeys('test');
```



 Puppeteer 流程

1. 光标应该 focus 到元素上
2. 键盘点击输入

```javascript
await page.focus('#kw');
await page.keyboard.sendCharacter('test');
```

在使用中可以多感受一下区别，会发现 Puppeteer 的使用会自然很多

## async/await

看官方的例子就可以看出来，几乎所有的操作都是异步的，如果坚持使用回调或者 Promise.then 写出来的代码会非常丑陋且难读，Puppeteer 官方推荐的也是使用高版本 Node 用 async/await 语法

```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://news.ycombinator.com', {waitUntil: 'networkidle'});
  await page.pdf({path: 'hn.pdf', format: 'A4'});

  await browser.close();
})();
```



## 查找元素

这是 UI 自动化测试最常用的功能了，Puppeteer 的处理也相当简单

1. page.$(selector)
2. page.$$(selector)

这两个函数分别会在页面内执行 `document.querySelector` 和 `document.querySelectorAll`，但返回值却不是 DOM 对象，如同 jQuery 的选择器，返回的是经过自己包装的 `Promise<ElementHandle>`，ElementHandle 帮我们封装了常用的 `click` 、`boundingBox` 等方法

## 获取 DOM 属性

我们写爬虫爬取页面图片列表，感觉可以通过 `page.$$(selector)` 获取到页面的元素列表，然后再去转成 DOM 对象，获取 src，然后并不行，想做对获取元素对应 DOM 属性的获取，需要用专门的 API

1. page.$eval(selector, pageFunction[, ...args])
2. page.$$eval(selector, pageFunction[, ...args])

大概用法

```javascript
const searchValue = await page.$eval('#search', el => el.value);
const preloadHref = await page.$eval('link[rel=preload]', el => el.href);
const html = await page.$eval('.main-container', e => e.outerHTML);
```

```javascript
const divsCounts = await page.$$eval('div', divs => divs.length);
```

值得注意的是如果 `pageFunction` 返回的是 Promise，那么 `page.$eval` 会等待方法 resolve

## evaluate

如果我们有一些及其个性的需求，无法通过 page.$() 或者 page.$eval() 实现，可以用大招——evaluate，有几个相关的 API

1. page.evaluate(pageFunction, …args)
2. page.evaluateHandle(pageFunction, …args): 
3. page.evaluateOnNewDocument(pageFunction, ...args)

这几个函数非常类似，都是可以在页面环境执行我们舒心的 JavaScript，区别主要在执行环境和返回值上

前两个函数都是在当前页面环境内执行，的主要区别在返回值上，第一个返回一个 [Serializable](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify#Description) 的 Promise，第二个返回值是前面提到的 ElementHandle 对象父类型 JSHandle 的 Promise

```javascript
const result = await page.evaluate(() => {
  return Promise.resolve(8 * 7);
});
console.log(result); // prints "56"

const aWindowHandle = await page.evaluateHandle(() => Promise.resolve(window));
aWindowHandle; // Handle for the window object. 相当于把返回对象做了一层包裹
```

`page.evaluateOnNewDocument(pageFunction, ...args)` 是在 browser 环境中执行，执行时机是文档被创建完成但是 script 没有执行阶段，经常用于修改 JavaScript 环境

## 注册函数

`page.exposeFunction(name, puppeteerFunction)` 用于在 window 对象注册一个函数，我们可以添加一个 `window.readfile` 函数

```javascript
const puppeteer = require('puppeteer');
const fs = require('fs');

puppeteer.launch().then(async browser => {
  const page = await browser.newPage();
  page.on('console', msg => console.log(msg.text));
  
  // 注册 window.readfile
  await page.exposeFunction('readfile', async filePath => {
    return new Promise((resolve, reject) => {
      fs.readFile(filePath, 'utf8', (err, text) => {
        if (err)
          reject(err);
        else
          resolve(text);
      });
    });
  });
  
  await page.evaluate(async () => {
    // use window.readfile to read contents of a file
    const content = await window.readfile('/etc/hosts');
    console.log(content);
  });
  await browser.close();
});
```



## 修改终端

Puppeteer 提供了几个有用的方法让我们可以修改设备信息

1. page.setViewport(viewport)
2. page.setUserAgent(userAgent)

```javascript
await page.setViewport({
  width: 1920,
  height: 1080
});

await page.setUserAgent('Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36');
```

page.emulateMedia(mediaType)：可以用来修改页面访问的媒体类型，但仅仅支持

1. screen
2. print
3. null：禁用 media emulation

page.emulate(options)：前面介绍的几个函数相当于这个函数的快捷方式，这个函数可以设置多个内容

1. viewport
   1. width
   2. height 
   3. deviceScaleFactor
   4. isMobile
   5. hasTouch
   6. isLandscape
2. userAgent

`puppeteer/DeviceDescriptors` 还给我们提供了几个大礼包

```javascript
const puppeteer = require('puppeteer');
const devices = require('puppeteer/DeviceDescriptors');
const iPhone = devices['iPhone 6'];

puppeteer.launch().then(async browser => {
  const page = await browser.newPage();
  await page.emulate(iPhone);
  await page.goto('https://www.google.com');
  // other actions...
  await browser.close();
});
```

## 键盘

1. keyboard.down
2. keyboard.up
3. keyboard.press
4. keyboard.type
5. keyboard.sendCharacter

```javascript
// 直接输入、按键
page.keyboard.type('Hello World!');
page.keyboard.press('ArrowLeft');

// 按住不放
page.keyboard.down('Shift');
for (let i = 0; i < ' World'.length; i++)
  page.keyboard.press('ArrowLeft');
page.keyboard.up('Shift');

page.keyboard.press('Backspace');
page.keyboard.sendCharacter('嗨');
```



## 鼠标 & 屏幕

1. mouse.click(x, y, [options]): options 可以设置
   1. button
   2. clickCount
2. mouse.move(x, y, [options]): options 可以设置
   1. steps
3. mouse.down([options])
4. mouse.up([options])
5. touchscreen.tap(x, y)

## 页面跳转控制

这几个 API 比较简单，不在展开介绍

1. page.goto(url, options)
2. page.goback(options)
3. page.goForward(options)

## 事件

Puppeteer 提供了对一些页面常见事件的监听，用法和 jQuery 很类似，常用的有

1. console：调用 console API
2. dialog：页面出现弹窗
3. error：页面 crash
4. load
5. pageerror：页面内未捕获错误



```javascript
page.on('load', async () => {
  console.log('page loading done, start fetch...');

  const srcs = await page.$$eval((img) => img.src);
  console.log(`get ${srcs.length} images, start download`);

  srcs.forEach(async (src) => {
    // sleep
    await page.waitFor(200);
    await srcToImg(src, mn);
  });

  await browser.close();

});
```



## 性能

通过 `page.getMetrics()` 可以得到一些页面性能数据

- `Timestamp` The timestamp when the metrics sample was taken.
- `Documents`  页面文档数
- `Frames` 页面 frame 数
- `JSEventListeners` 页面内事件监听器数
- `Nodes`  页面 DOM 节点数
- `LayoutCount` 页面 layout 数
- `RecalcStyleCount` 样式重算数
- `LayoutDuration` 页面 layout 时间
- `RecalcStyleDuration`  样式重算时长
- `ScriptDuration` script 时间
- `TaskDuration`  所有浏览器任务时长
- `JSHeapUsedSize`  JavaScript 占用堆大小
- `JSHeapTotalSize`  JavaScript 堆总量

```javascript
{ 
  Timestamp: 382305.912236,
  Documents: 5,
  Frames: 3,
  JSEventListeners: 129,
  Nodes: 8810,
  LayoutCount: 38,
  RecalcStyleCount: 56,
  LayoutDuration: 0.596341000346001,
  RecalcStyleDuration: 0.180430999898817,
  ScriptDuration: 1.24401400075294,
  TaskDuration: 2.21657899935963,
  JSHeapUsedSize: 15430816,
  JSHeapTotalSize: 23449600 
}
```

## 最后

本文知识介绍了部分常用的 API，全部的 API 可以在 [github](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#) 上查看，由于 Puppeteer 还没有发布正式版，API 迭代比较迅速，在使用中遇到问题也可以在 issue 中反馈。

在 0.11 版本中只有 `page.$eval` 并没有 `page.$$eval`，使用的时候只能通过 `page.evaluate`，通过大家的反馈，在 0.12 中已经添加了该功能，总体而言 Puppeteer 还是一个十分值得期待的 Node headless API

## 参考

[Getting Started with Headless Chrome](https://developers.google.com/web/updates/2017/04/headless-chrome)

[无头浏览器 Puppeteer 初探](https://juejin.im/post/59e5a86c51882578bf185dba)

[Getting started with Puppeteer and Chrome Headless for Web Scraping](https://github.com/emadehsan/thal)
