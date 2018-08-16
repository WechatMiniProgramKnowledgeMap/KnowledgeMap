- 首先我们在pages/xxxPage/main.js中会调用``Vue.$mount``方法
```
import Vue from 'vue'
import App from './index'

const app = new Vue(App)
app.$mount()
```
上面这个方法会走到``src/platforms/mp/runtime/index.js``中的``$mount``方法：
```
Vue.prototype.$mount = function (el, hydrating) {
  // el = el && inBrowser ? query(el) : undefined
  // return mountComponent(this, el, hydrating)

  // 初始化小程序生命周期相关
  const options = this.$options

  if (options && (options.render || options.mpType)) {
    const { mpType = 'page' } = options
    return this._initMP(mpType, () => {
      return mountComponent(this, undefined, undefined)
    })
  } else {
    return mountComponent(this, undefined, undefined)
  }
}
```
# mpvue-loader/lib/loader.js
- 1.将vue文件解析成模板对象
```
var parts = parse(content, fileName, this.sourceMap)
```
得到的结果大致如下：
```
{
	"template": {
		"type": "template",
		"content": "\n<view class=\"container-view\">\n  <home-quotation-item v-for=\"(stock, index) in stockList\" :key=\"index\" :quotationItem=\"stock\" @itemClicked=\"itemClicked\"></home-quotation-item>\n</view>\n",
		"start": 10,
		"attrs": {},
		"end": 391
	},
	"script": {
		"type": "script",
		"content": "//\n//\n//\n//\n//\n//\n//\n//\n//\n//\n\nimport homeQuotationItem from '@/components/homeQuotationItem'\n\nexport default {\n  data () {\n    return {\n    }\n  },\n  props: {\n    reason: String,\n    stockList: Array\n  },\n  components: {\n    homeQuotationItem\n  },\n  methods: {\n    itemClicked (quotationItem) {\n      this.$emit('itemViewClicked', quotationItem)\n    }\n  }\n}\n",
		"start": 412,
		"attrs": {},
		"end": 740,
		...
	},
	"styles": [{
		"type": "style",
		"content": "\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n.container-view\n  width :100%\n  background :white\n  margin-bottom :18rpx\n\n.section-headView\n  width :100%\n  height :100%\n  display :flex\n  align-items :center\n  margin-left :24rpx\n  padding :18rpx 0\n",
		"start": 779,
		"attrs": {
			"lang": "stylus",
			"scoped": true
		},
		"lang": "stylus",
		"scoped": true,
		"end": 1193,
		...
	}],
	"customBlocks": []
}
```
2.调用mpvue-loader/lib/template-compiler/index.js导出的接口并传入上面得到的html模板：
```
var templateCompilerPath = normalize.lib('template-compiler/index')
...
var defaultLoaders = {
  html: templateCompilerPath + templateCompilerOptions,
  css: options.extractCSS
    ? getCSSExtractLoader()
    : styleLoaderPath + '!' + 'css-loader' + cssLoaderOptions,
  js: hasBuble ? ('buble-loader' + bubleOptions) : hasBabel ? babelLoaderOptions : ''
}

// check if there are custom loaders specified via
// webpack config, otherwise use defaults
var loaders = Object.assign({}, defaultLoaders, options.loaders)
```

# mpvue-loader/lib/template-compiler/index.js
1. 调用mpvue/packages/mpvue-template-compiler/build.js的compile接口：
```
var compiled = compile(html, compilerOptions)
```
compile方法生产下面的ast(Abstract Syntax Tree)模板，render函数和staticRenderFns
```
{
  ast: {
    type: 1,
    tag: 'view',
    attrsList: [],
    attrsMap: {
      class: 'container-bg'
    },
    parent: undefined,
    children: [{
        type: 1,
        tag: 'view',
        attrsList: [],
        attrsMap: {
          class: 'mask-view',
          'v-if': 'isShow'
        },
        parent: {
          type: 1,
          tag: 'view',
          attrsList: [],
          attrsMap: {
            class: 'container-bg'
          },
          parent: undefined,
          children: [
            ...
          ],
          plain: false,
          staticClass: '"container-bg"',
          static: false,
          staticRoot: false
        },
        children: [],
        if: 'isShow',
        ifConditions: [{
          exp: 'isShow',
          block: [Circular]
        }],
        plain: false,
        staticClass: '"mask-view"',
        static: false,
        staticRoot: false,
        ifProcessed: true
      },
      ...
    ],
    plain: false,
    staticClass: '"container-bg"',
    static: false,
    staticRoot: false
  },
  render: 'with(this){return _c(\'view\',{staticClass:"container-bg"},[(isShow)?_c(\'view\',{staticClass:"mask-view"}):_e(),_v(" "),_c(\'view\',{staticClass:"head-view"},[_c(\'view\',{staticClass:"left-view"},[_c(\'view\',{staticClass:"line-view"}),_v(" "),_c(\'view\',{staticClass:"top-dateView"},[_v(_s(rankDate)+" "+_s(listNum)+"条数据")])]),_v(" "),_c(\'form\',{attrs:{"report-submit":"true","eventid":\'1\'},on:{"submit":onSubscribeClick}},[_c(\'button\',{staticClass:"right-view",attrs:{"form-type":"submit","open-type":"getUserInfo","eventid":\'0\'},on:{"getuserinfo":onGetUserInfo}},[_c(\'image\',{staticClass:"subscribe-icon",attrs:{"mode":"aspectFit","src":"/static/images/subscribe-icon.png"}}),_v(" "),_c(\'view\',{staticClass:"subscribe-title"},[_v("订阅")])])],1)],1),_v(" "),_c(\'view\',{staticClass:"home-container"},_l((lists),function(item,index){return _c(\'home-quotation-view\',{key:index,attrs:{"reason":item.reason,"stockList":item.list,"eventid":\'2-\'+index,"mpcomid":\'0-\'+index},on:{"itemViewClicked":itemViewClicked}})})),_v(" "),_c(\'view\',{staticClass:"footer-view"},[_v("没有更多")])])}',
  staticRenderFns: [],
  errors: [],
  tips: []
}
```
其中的render函数运行的结果是返回``VNode``对象，其实``render``函数应该长下面这样：
```
(function() {
  with(this){
    return _c('div',{   //创建一个 div 元素
      attrs:{"id":"app"}  //div 添加属性 id
      },[
        _m(0),  //静态节点 header，此处对应 staticRenderFns 数组索引为 0 的 render 函数
        _v(" "), //空的文本节点
        (message) //三元表达式，判断 message 是否存在
         //如果存在，创建 p 元素，元素里面有文本，值为 toString(message)
        ?_c('p',[_v("\n    "+_s(message)+"\n  ")])
        //如果不存在，创建 p 元素，元素里面有文本，值为 No message. 
        :_c('p',[_v("\n    No message.\n  ")])
      ]
    )
  }
})
```
其中的``_c``就是vue对象的``createElement``方法 (创建元素)，``_m``是``renderStatic``（渲染静态节点），``_v`` 是 
``createTextVNode``（创建文本dom），``_s`` 是 ``toString`` （转换为字符串）
```
// src/core/instance/render.js
export function initRender (vm: Component) {
  ...
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
  ...
}

...
Vue.prototype._s = toString
...
Vue.prototype._m = renderStatic
...
Vue.prototype._v = createTextVNode
...
```
2. 调用compileWxml方法生产wxml模板，这个方法最终会调用 mpvue/packages/mpvue-template-compiler/build.js的compileToWxml方法将第一步compile出来的模板转成小程序的wxml模板
```
// mpvue-loader/lib/template-compiler/index.js
compileWxml.call(this, compiled, html)
```
总结可以得出mpvue-loader 和 mpvue/packages/mpvue-template-compiler，会在编译期，将html模板预编译成小程序的模板，而小程序本身不支持动态增删改查 wxml 节点信息，所以mpvue没有办法做到组件自定义 render函数。