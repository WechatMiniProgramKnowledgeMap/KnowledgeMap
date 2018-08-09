- [开言](#开言)
- [小程序架构](#小程序架构)

# 开言

> 开发中使用库当然没问题，但前提是不要仅仅只知道怎么使用，更要知道它的工作原理。如果不能超越这些库，那在这些库变成你的拐杖之时，你也会随之“残废”。

知其然，并知其所以然。时刻要有一颗追本溯源的心。

# 小程序架构

![架构图](https://pic4.zhimg.com/80/v2-2b982ccd90101fab6b6c5994abfc66b1_hd.jpg)

微信小程序的框架包含两部分View视图层、App Service逻辑层。View层用来渲染页面结构，AppService层用来逻辑处理、数据请求、接口调用。

它们在**两个线程里**运行。

它们在**两个线程里**运行。

它们在**两个线程里**运行。

视图层和逻辑层通过系统层的JSBridage进行通信，逻辑层把数据变化通知到视图层，触发视图层页面更新，视图层把触发的事件通知到逻辑层进行业务处理。

**视图层使用WebView渲染，逻辑层使用在iOS中使用自带的JSCore运行，在android中使用腾讯的x5内核运行。开发工具使用nw.js同时提供了视图层和逻辑层的运行环境。**

我们不能在代码里使用BOM和DOM是因为根本没有，另一方面也不希望JS代码直接操作视图。

在开发工具中remote-helper.js中找到了这样的代码：

```js
const vm = require("vm");

const vmGlobal = {
    require: undefined,
    eval: undefined,
    process: undefined,
    setTimeout(...args) {
        const timer = global.setTimeout.call(global, ...args);
        timers[++timerCount] = timer;
        if (timerCount % 5 === 0) {
            for (const key in timers) {
                const item = timers[key];
                if (item && !item._repeat && item._called && item._destroyed) {
                    delete timers[key];
                }
            }
        }
        return timerCount;
    },
    clearTimeout(id) {
        const timer = timers[id];
        if (timer) {
            clearTimeout(timer);
            delete timers[id];
        }
    },
    setInterval(...args) {
        const timer = global.setInterval.call(global, ...args);
        timers[++timerCount] = timer;
        if (timerCount % 5 === 0) {
            for (const key in timers) {
                const item = timers[key];
                if (item && !item._repeat && item._called && item._destroyed) {
                    delete timers[key];
                }
            }
        }
        return timerCount;
    },
    clearInterval(id) {
        const timer = timers[id];
        if (timer) {
            clearInterval(timer);
            delete timers[id];
        }
    },
    console: (() => {
        let consoleClone = Object.create(Object.getPrototypeOf(console));
        const names = Object.getOwnPropertyNames(console);
        for (const key of names) {
            try {
                const desc = Object.getOwnPropertyDescriptor(console, key);
                if (desc) {
                    const newDesc = {
                        configurable: false,
                    };
                    if (desc.hasOwnProperty('writable')) {
                        newDesc.writable = false;
                    }
                    Object.defineProperty(consoleClone, key, Object.assign({}, desc, newDesc));
                }
                else {
                    consoleClone[key] = console[key];
                }
            }
            catch (e) {
                consoleClone = Object.assign(Object.create(Object.getPrototypeOf(console)), console);
                Object.freeze(consoleClone);
                break;
            }
        }
        return consoleClone;
    })()
};
const jsVm = vm.createContext(vmGlobal);
// 省略大量代码...
function loadCode(filePath, sourceURL, content) {
    let ret;
    try {
        const script = typeof content === 'string' ? content : fs.readFileSync(filePath, 'utf-8').toString();
        ret = vm.runInContext(script, jsVm, {
            filename: sourceURL,
        });
    }
    catch (e) {
        // something went wrong in user code
        console.error(e);
    }
    return ret;
}
```



这样的分层设计显然是有意为之的，它的中间层完全控制了程序对于界面进行的操作， 同时对于传递的数据和响应时间也能做到监控。一方面程序的行为受到了极大限制， 另一方面微信可以确保他们对于小程序内容和体验有绝对的控制。

这样的结构也说明了小程序的动画和绘图 API 被设计成生成一个最终对象而不是一步一步执行的样子， 原因就是 json格式的数据传递和解析相比与原生 API 都是损耗不菲的，如果频繁调用很可能损耗过多性能，进而影响用户体验。

**需要注意的是动画需要绑定在data上，而绘图却不用。你觉得是为什么呢？**

```js
var context = wx.createCanvasContext('firstCanvas')
    
context.setStrokeStyle("#00ff00")
context.setLineWidth(5)
context.rect(0, 0, 200, 200)
context.stroke()
context.setStrokeStyle("#ff0000")
context.setLineWidth(2)
context.moveTo(160, 100)
context.arc(100, 100, 60, 0, 2 * Math.PI, true)
context.moveTo(140, 100)
context.arc(100, 100, 40, 0, Math.PI, false)
context.moveTo(85, 80)
context.arc(80, 80, 5, 0, 2 * Math.PI, true)
context.moveTo(125, 80)
context.arc(120, 80, 5, 0, 2 * Math.PI, true)
context.stroke()
context.draw()
```



```Js
Page({
  data: {
    animationData: {}
  },
  onShow: function(){
    var animation = wx.createAnimation({
      duration: 1000,
  	  timingFunction: 'ease',
    })

    this.animation = animation
    
    animation.scale(2,2).rotate(45).step()
    
    this.setData({
      animationData:animation.export()
    })
  }
})
```



