## 原因：
目前的组件是使用小程序的 template 标签实现的，给组件指定的class和style是挂载在template标签上，而template 标签不支持 class 及 style 属性。

## 解决方案：
给自定义组件指定的标签直接挂载在子元素上就好了。
- 在自定义组件上绑定class到一个props属性上

```js
 // 组件ComponentA.vue
 <template>
  <div class="container" :class="pClass">
    ...
  </div>
</template>
<script>
    export default {
    props: {
      pClass: {
        type: String,
        default: ''
      }
    }
  }
</script>
```
 
```js
 // PageB.vue
<template>
    <component-a :pClass="cusComponentAClass"  />
</template>
<script>
data () {
    return {
      cusComponentAClass: 'a-class b-class'
    }
  }
</script>

<style lang="stylus" scoped>
  .a-class
    border red solid 2rpx
  .b-class
    margin-right 20rpx
</style>
```
 但是这样会有问题就是style加上scoped之后，编译模板生成的代码是下面这样的：
 
```js
 .a-class.data-v-8f1d914e {
   border: #f00 solid 2rpx;
 }
 .b-class.data-v-8f1d914e {
   margin-right 20rpx
 }
```
 所以想要这些组件的class生效就不能使用scoped的style，改成下面这样，最好自己给a-class和b-class加前缀以防其他的文件引用这些样式：
 
```js
 <style lang="stylus">
  .a-class
    border red solid 2rpx
  .b-class
    margin-right 20rpx
</style>

<style lang="stylus" scoped>
  .other-class
    border red solid 2rpx
    
   ...
</style>
```
- 在定义组件上绑定style属性到一个props属性上：
 
```js
 // 组件ComponentA.vue
 <template>
  <div class="container" :style="pStyle">
    ...
  </div>
</template>
<script>
    export default {
    props: {
      pStyle: {
        type: String,
        default: ''
      }
    }
  }
</script>
```
 
```js
 // PageB.vue
<template>
    <component-a :pStyle="cusComponentAStyle"  />
</template>
<script>
const cusComponentAStyle = 'border:red solid 2rpx; margin-right:20rpx;'
data () {
    return {
      cusComponentAStyle
    }
  }
</script>

<style lang="stylus" scoped>
  ...
</style>
```
 
也可以通过定义styleObject，然后通过工具函数转化为styleString，如下所示：
 
```js
const bstyle = {
  border: 'red solid 2rpx',
  'margin-right': '20rpx'
}
let arr = []
for (let [key, value] of Object.entries(bstyle)) {
  arr.push(`${key}: ${value}`)
}

const cusComponentAStyle = arr.join('; ')
```
 
- 当然自定义组件确定只会改变某个css样式，通过pros传入但单个样式的值，然后通过:style绑定肯定没问题：

```js
// 组件ComponentA.vue
 <template>
  <div class="container" :style="{'background-color': backgroundColor}">
    ...
  </div>
</template>
<script>
    export default {
    props: {
      backgroundColor: {
        type: String,
        default: 'yellow'
      }
    }
  }
</script>
```

```js
 // PageB.vue
<template>
    <component-a backgroundColor="red"  />
</template>

```