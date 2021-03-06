---
title: react-motion翻译及学习
date: 2018-08-14 14:58:28
tags:
  - react
  - react-motion
  - 翻译
categories: 前端
---

# react-motion文档及学习

之前没怎么重视过动画，所以做出来的东西体验不是特别好，所以回过来学一些动画库，然后发现 `react-motion` 这个库的例子真棒，就学一下这个库的 `API` ，在 `github` 上也只有[英文版文档](https://github.com/chenglou/react-motion)，好在文档内容不算长，所以准备大致翻译一下。

## 这个库解决了什么问题？

在95%的动画组件的应用场景下，我们不必采用硬编码的缓动曲线和持续时间。给你的 `UI` 组件设置一个刚度值和阻尼值，然后让物理魔法来处理剩下的事情。这样就不必担心出现动画中断之类的小问题。这也大大地简化了 `API`。

这个库同时还为 `React` 的 `TransitionGroup` 提供了一个更强大的替代 `API` 。

## API

Exports:

* spring
* Motion
* StaggeredMotion
* TransitionMotion
* presets

## 辅助函数

> -spring: (val: number, config?: SpringHelperConfig) => OpaqueConfig

与相关的组件一起用，明确如何动画到目标值，比如 `spring(10, {stiffness: 120, damping: 17})`意味着以 **120** 的 *刚度* 和 **17** 的 *阻尼*运动到*目标值* 10。

* `val` ：目标值
* `config` : 可选，作进一步调整。可能的属性值：
  * `stiffness` : 刚度，可选，默认 `170`
  * `damping` : 阻尼，可选， 默认 `26`
  * `precision` : 可选，默认 `0.01`， 指定内插值的舍入和速度，内部值，一般外部不用去改变改值。

> Presets for {stiffness, damping}

大部分是像 `spring(10, presets.wobbly)` 、 `spring(20, {...presets.gentle})` 这样子使用 spring 的预设配置。

## `<Motion/>`

一般的动画组件

### `<Motion/>` 用法

```js
/**
 * interpolation -> 插值
 * Motion组件的写法比较特殊，传了一个函数当作 Motion 的 chlidren属性
*/
<Motion defaultStyle={{x: 0}} style={{x: spring(10)}}>
    {interpolatingStyle => <div style={interpolatingStyle} />}
</Motion>
```

### `<Motion/>` 属性(props)

-style: Style

必须。 `Style` 是一个对象类型映射到上面的 `spring` 返回的数字或者是 `OpaqueConfig` 对象。在组件整个存活期间必须保持相同的值。值的意义：

* `spring` 返回的 `OpaqueConfig` : 插值到 `x`
* `spring` 返回的数字 `x`: 跳到 `x`， 不插入

-defaultStyle?: PlainStyle

可选。`PlainStyle` 类型映射到数字，和上面的 `style` 相同键的对象，值是初始化要插入的数据。注意，在后续的渲染过程中这个属性会被忽略。从当前值插入的目标值是由 `style` 决定。

-children: (interpolatedStyle: PlainStyle) => ReactElement

必须，且是函数。

* `interpolatedStyle`: 返回给你被插入值的 `style` 对象。举个例子： 如果你写了 `style={...xxx}`, 然后就会在函数里接收到 `interpolatedStyle` 对象，在某一时间，可能拿到这样的数据 `{x:5.2, y: 12.1}`,就可以把这些数据用到你的 `div` 上或者其他地方。
* `Return` 一定要返回一个 `React element`，这样 `Motion` 才能渲染你的组件。

-onRest?: () => void

可选。动画休息的时候触发的回调函数。

## `<StaggeredMotion/>`

交错的动画组件。

创建一个彼此之间依赖的集合（固定长度）的动画，达到*自然*、*弹性*、*惊人*的效果。这比对一系列运动硬编码延迟的方式（不太自然的动画效果）更加有效。

### `<StaggeredMotion/>` 用法

```js
<StaggeredMotion
  defaultStyles={[{h: 0}, {h: 0}, {h: 0}]}
  styles={prevInterpolatedStyles => prevInterpolatedStyles.map((_, i) => {
    return i === 0
      ? {h: spring(100)}
      : {h: spring(prevInterpolatedStyles[i - 1].h)}
  })}>
  {interpolatingStyles =>
    <div>
      {interpolatingStyles.map((style, i) =>
        <div key={i} style={{border: '1px solid', height: style.h}} />)
      }
    </div>
  }
</StaggeredMotion>
```

### `<StaggeredMotion/>` 属性(props)

`-styles:(previousInterpolatedStyles: ?Array<PlainStyle>) => Array<style>`

必须,函数。**不要忘记`"s"`**

* `previousInterpolatedStyles` 前一个插入值的 `styles` ,(第一次渲染的时候是 `undefined` ,除非提供了 `defaultStyles`)

* Return 一定要返回包含*目的值*的 `styles` 数组,举个例子:[{x: spring(10), x: spring(20)}]

-defaultStyles?: Array<PlainStyle>

可选。和 `Motion` 的 `defaultStyle` 类似，但是是一个数组。

-children: (interpolatedStyles: Array<PlainStyle>) => ReactElement

必须，函数。和 `Motion` 的 `children` 类似, 但是接受一个插入值的数组作为函数的参数。举个例子：`[{x: 5}, {x: 6.4}, {x: 8.1}]`。

(没有 `noRest`, 我们没有发现有什么意义在 `StaggeredMotion` 中加入 `noReset`)

## `<TransitionMotion>`

帮助你创建 `mounting` 和 `unmounting` 时的动画。

### `<TransitionMotion>` 用法

有 `a`、`b`、`c` 三个项目，有各自的 `style` 配置，

---

![Edvard Munch – The Scream](react-motion翻译及学习/944233120.jpg)

> Edvard Munch – The Scream. ver. 1893