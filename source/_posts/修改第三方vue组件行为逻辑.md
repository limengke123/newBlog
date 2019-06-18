---
title: 修改第三方vue组件行为逻辑
date: 2019-06-18 21:59:04
tags:
  - vue
categories: 前端
---

# 修改第三方 vue 组件行为逻辑

在日常开发的时候难免会遇到组件库的组件内部的一些逻辑或者部分渲染的内容不符合真正业务的需要，这时候去重写这么一个组件就太费劲，那能不能在这个组件的基础上去偷偷修改这个组件的一些行为方式呢？

真实的案例就是最近一个项目中用到了公司内部的 `Upload` 组件，在图片上传失败之后，`Upload` 组件会显示出 **文件上传失败，请重新上传**，由于某种特殊的要求需要把这里的文案改成 **上传失败** ，当然这个 `Upload` 组件并没有提供这么一个文案的 `props` 去配置。

那么如何在这个组件的基础上去偷偷修改这里显示的文案呢？

## 思路

这里先铺垫一点 `vue` 的知识，我们日常开发所写的 `*.vue` 的 `sfc` 组件实际上 `export default` 出来的是一个 *选项对象*，其中 `template` 里面的内容被 `vue-loader` 编译成了 `render` 函数，选项对象里面包含了诸如 `data`，`methods`，`render` 之类的属性，*选项对象* 本质上就是一个普通的对象。

在调用组件的时候，诸位应该注意到需要在使用之前需要把导出的这个组件通过 `Vue.component` 去注册一次，这个步骤实际上是把 *选项对象* 真正地包装成了一个 `vue` 的构造器，观察官方文档：

```javascript
// 注册组件，传入一个扩展过的构造器
Vue.component('my-component', Vue.extend({ /* ... */ }))

// 注册组件，传入一个选项对象 (自动调用 Vue.extend)
Vue.component('my-component', { /* ... */ })

// 获取注册的组件 (始终返回构造器)
var MyComponent = Vue.component('my-component')
```

`component` 函数签名的第二参数可以传入一个已经扩展过的构造器或者是一个选项对象就能生成一个 `vue` 的构造函数。

这里注意 `Vue.extend` 函数:

> 使用基础 Vue 构造器，创建一个“子类”。参数是一个包含组件选项的对象。

```javascript
// 官方例子
// 创建构造器
var Profile = Vue.extend({
  template: '<p>{{firstName}} {{lastName}} aka {{alias}}</p>',
  data: function () {
    return {
      firstName: 'Walter',
      lastName: 'White',
      alias: 'Heisenberg'
    }
  }
})
// 创建 Profile 实例，并挂载到一个元素上。
new Profile().$mount('#mount-point')
```

结果如下：`<p>Walter White aka Heisenberg</p>`。

针对如何修改第三方组件的逻辑这个问题，这里我们可以尝试去扩展 `Upload` 这个组件，同时在扩展的子类上面加上一些我们需要的一些逻辑去覆盖掉组件库的逻辑。

## 实践

```typescript
import { Upload } from 'so-ui'
import Vue, {CreateElement, VNode} from 'vue'

interface IUploadHtml extends HTMLDivElement {
    clearFiles: () => void
}

const myUpload = Vue.extend({
    name: 'my-upload',
    props: {
        ...Upload.props,
        clear: false
    },
    render (createElement: CreateElement ): VNode {
        return createElement(Upload, {
            props: {
                ...this.$props,
                onError: this.handleError
            },
            on: this.$listeners,
            attrs: this.$attrs,
            ref: 'uploads'
        })
    },
    watch: {
        clear() {
            (this.$refs.uploads as IUploadHtml).clearFiles()
        } 
    },
    methods: {
        handleError() {
            setTimeout(() => {
                const upload = this.$refs.uploads as Vue;
                (upload.$el.querySelector('.so-upload-list__item:nth-child(1) p') as HTMLElement).innerText = '正在上传';
                (upload.$el.querySelector('.so-upload-list__item:nth-child(2) p') as HTMLElement).innerText = '上传失败'
            }, 300)
        }
    }
})

export default myUpload

```

这里我们尝试去扩展 `Upload` 这个组件，定义了一个 `my-upload` 的包裹组件，在这个包裹组件的 `render` 函数里，直接去调用渲染 `Upload` 组件，在 `Upload` 组件上监听了上传失败的事件，同时通过 `$refs` 找到 `dom` 节点，直接去修改生成文案节点内容。

熟悉 `react` 的 `hoc` 模式的能发现，这种方式其实是 `hoc` 很类似，在原来的组件基础上去做一些操作。但是和 `react` 类似，因为包裹了一层，需要把 `props`、 `事件` 透传给真正的 `Upload`。

```js
return createElement(Upload, {
            props: {
                ...this.$props,
                onError: this.handleError
            },
            on: this.$listeners,
            attrs: this.$attrs,
            ref: 'uploads'
        })
```

## 总结

通过 `Vue.extends`，`vue` 在某种程度上可以去修改和覆盖一些第三方组件逻辑，这里的例子举的不是特别好，只是通过ref去找到dom节点改掉了文本，实际上这种类似与 `hoc` 的功能配合 `mixin` 应该有更强大的应用。
