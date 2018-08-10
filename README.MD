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

在mac下使用js-beautify对微信开发工具@v1.02.1808080代码批量格式化：

```Shell
cd /Applications/wechatwebdevtools.app/Contents/Resources/package.nw
find . -type f -name '*.js' -not -path "./node_modules/*" -not -path -exec js-beautify -r -s 2 -p -f '{}' \;
```



在js/extensions/appservice/index.js中找到：

```js
	267: function(a, b, c) {
    const d = c(8),
      e = c(227),
      f = c(226),
      g = c(228),
      h = c(229),
      i = c(230);
    var j = window.__global.navigator.userAgent,
      k = -1 !== j.indexOf('game');
    k || i(), window.__global.getNewWeixinJSBridge = (a) => {
      const {
        invoke: b
      } = f(a), {
        publish: c
      } = g(a), {
        subscribe: d,
        triggerSubscribeEvent: i
      } = h(a), {
        on: j,
        triggerOnEvent: k
      } = e(a);
      return {
        invoke: b,
        publish: c,
        subscribe: d,
        on: j,
        get __triggerOnEvent() {
          return k
        },
        get __triggerSubscribeEvent() {
          return i
        }
      }
    }, window.WeixinJSBridge = window.__global.WeixinJSBridge = window.__global.getNewWeixinJSBridge('global'), window.__global.WeixinJSBridgeMap = {
      __globalBridge: window.WeixinJSBridge
    }, __devtoolsConfig.online && __devtoolsConfig.autoTest && setInterval(() => {
      console.clear()
    }, 1e4);
    try {
      var l = new window.__global.XMLHttpRequest;
      l.responseType = 'text', l.open('GET', `http://${window.location.host}/calibration/${Date.now()}`, !0), l.send()
    } catch (a) {}
  }
```

在js/extensions/gamenaitveview/index.js中找到：

```js
  299: function(a, b, c) {
    'use strict';
    Object.defineProperty(b, '__esModule', {
      value: !0
    });
    var d = c(242),
      e = c(241),
      f = c(243),
      g = c(244);
    window.WeixinJSBridge = {
      on: d.a,
      invoke: e.a,
      publish: f.a,
      subscribe: g.a
    }
  },
```

在js/extensions/pageframe/index.js中找到：

```js
317: function(a, b, c) {
    'use strict';

    function d() {
      window.WeixinJSBridge = {
        on: e.a,
        invoke: f.a,
        publish: g.a,
        subscribe: h.a
      }, k.a.init();
      let a = document.createEvent('UIEvent');
      a.initEvent('WeixinJSBridgeReady', !1, !1), document.dispatchEvent(a), i.a.init()
    }
    Object.defineProperty(b, '__esModule', {
      value: !0
    });
    var e = c(254),
      f = c(253),
      g = c(255),
      h = c(256),
      i = c(86),
      j = c(257),
      k = c.n(j);
    'complete' === document.readyState ? d() : window.addEventListener('load', function() {
      d()
    })
  },
```





我们都看到了WeixinJSBridge的定义。分别都有`on`,`invoke`,`publish`,`subscribe`这个几个关键方法。

拿`invoke`举例，在js/extensions/appservice/index.js中发现这段代码：

```js
f (!r) p[b] = s, f.send({
    command: 'APPSERVICE_INVOKE',
    data: {
        api: c,
        args: e,
        callbackID: b
    }
});
```

在js/extensions/pageframe/index.js中发现这段代码：

```js
g[d] = c, e.a.send({
    command: 'WEBVIEW_INVOKE',
    data: {
        api: a,
        args: b,
        callbackID: d
    }
})

```

简单的分析得知：字段command用来区分行为，invoke用来调用native的api。在不同的来源要使用不同的前缀。data里面包含api名，参数。另外callbackID指定接受回调的方法句柄。appservice和webview使用的通信协议是一致的。





我们不能在代码里使用BOM和DOM是因为根本没有，另一方面也不希望JS代码直接操作视图。

在开发工具中remote-helper.js中找到了这样的代码：

```js
const vm = require("vm");

const vmGlobal = {
    require: undefined,
    eval: undefined,
    process: undefined,
    setTimeout(...args) {
        //...省略代码
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
        //...省略代码
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
        //...省略代码
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



# 下载小程序完整包

![Download](https://pic4.zhimg.com/80/v2-d6deb274157ccffbc5cbc762503e9504_hd.jpg)



# View - WXML

WXML（WeiXin Markup Language）

- 支持数据绑定
- 支持逻辑算术、运算
- 支持模板、引用
- 支持添加事件（bindtap）

![WXML](https://pic3.zhimg.com/80/v2-e0a34d00890cab73c79d137edd1377a3_hd.jpg)

wxml编译器：wcc  把wxml文件 转为 js 

执行方式：wcc index.wxml



# View - WXSS

WXSS(WeiXin Style Sheets)

![WXSS](https://pic2.zhimg.com/80/v2-3829f8c15260cd0cbcbadfab3446ad65_hd.jpg)

wxss编译器：wcsc 把wxss文件转化为 js 

执行方式： wcsc index.wxss



## 支持大部分CSS特性

亲测包含但不限于如下内容：

- Transition
- Animation
  - Keyframes
- border-radius
- calc()
- 选择器，除了[官方文档](https://developers.weixin.qq.com/miniprogram/dev/framework/view/wxss.html)列出的，其实还支持
  - element>element
  - element+element



建议css3的特性都可以做一下尝试。



## 尺寸单位rpx，可根据屏幕宽度自适应

rpx（responsive pixel）: 可以根据屏幕宽度进行自适应。规定屏幕宽为750rpx。公式:

```Js
const dsWidth = 750

export const screenHeightOfRpx = function () {
  return 750 / env.screenWidth * env.screenHeight
}

export const rpxToPx = function (rpx) {
  return env.screenWidth / 750 * rpx
}

export const pxToRpx = function (px) {
  return 750 / env.screenWidth * px
}

```

| 设备         | rpx换算px (屏幕宽度/750) | px换算rpx (750/屏幕宽度) |
| ------------ | ------------------------ | ------------------------ |
| iPhone5      | 1rpx = 0.42px            | 1px = 2.34rpx            |
| iPhone6      | 1rpx = 0.5px             | 1px = 2rpx               |
| iPhone6 Plus | 1rpx = 0.552px           | 1px = 1.81rpx            |

## 样式导入

使用`@import`语句可以导入外联样式表，`@import`后跟需要导入的外联样式表的相对路径，用`;`表示语句结束。



## 内联样式

静态的样式统一写到 class 中。style 接收动态的样式，在运行时会进行解析，**请尽量避免将静态的样式写进 style 中，以免影响渲染速度**。



## 全局样式与局部样式

定义在 app.wxss 中的样式为全局样式，作用于每一个页面。在 page 的 wxss 文件中定义的样式为局部样式，只作用在对应的页面，并会覆盖 app.wxss 中相同的选择器。



# View - Component

小程序提供了一系列组件用于开发业务功能，按照功能与HTML5的标签进行对比如下：

![Component](https://pic3.zhimg.com/80/v2-480318a5bae828e51e8f05c1ea1921fa_hd.jpg)

小程序的组件基于Web Component标准

使用Polymer框架实现Web Component



# View - Native Component

目前Native实现的组件有

- <cavnas/>

- <video/>

- <map/>

- <textarea/>

  ![Native Component](https://pic1.zhimg.com/80/v2-f0e838350357658699aeeed7dad74048_hd.jpg)

Native组件层在WebView层之上。这目前带来了一些问题：