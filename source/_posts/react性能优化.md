---
title: react性能优化
date: 2019-04-07 11:43:23
tags:
  - react
categories: 
  - 前端
---

# react 性能优化

为什么需要优化？

> `react` 的子组件在父组件渲染的时候，在默认的情况下，不论这个父组件传递给子组件的`props` 是否发生了改变，这个子组件就会去重新渲染，然后这种渲染是完全没有必要的，所以假想我们的 `react` 工程完全没有去针对这个问题作出优化，只要组件触发了渲染，那么整个子组件都将触发一次渲染，这就会造成性能上的问题。

## 一个 counter 的例子

这里举一个简单的计数器例子来看一下这个问题。

### Counter组件

```jsx harmony
class Counter extends React.Component {
    state = {
        count: 0
    }
    increase = () => {
        this.setState(state => ({
            count: state.count + 1
        }))
    }
    decrease = () => {
        this.setState(state => ({
            count: state.count - 1
        }))
    }
    render() {
        const { count } = this.state
        console.log('count container page render')
        return (
            <div className={styles.home}>
                <Count count={count} />
                <div>
                    <Button onClick={this.decrease} text="-" />
                    <Button onClick={this.increase} text="+" />
                </div>
            </div>
        )
    }
}
```

### Count组件

依赖于Counter组件传递count到这里显示数字:

```jsx harmony
class Count extends React.Component {
    static propTypes = {
        count: PropTypes.number
    }
    render() {
        console.log('count number render')
        const { count } = this.props
        return <div className={styles.count}>{count}</div>
    }
}
```

### Button组件

```jsx harmony
class Button extends React.Component {
    static propTypes = {
        text: PropTypes.string,
        onClick: PropTypes.func
    }
    render() {
        console.log('count button render')
        const { text, onClick: onClickProps } = this.props
        return (
            <button type="button" onClick={onClickProps}>
                {text}
            </button>
        )
    }
}
```

`Counter` 组件下有两个子组件，`Count` 通过props拿到Counter组件中的state.count，给两个button组件分别加上增加和减少Counter组件的state.count的回调函数。同时了方便观察，我分别在三个组件的render函数中加上了一句打印。

点击增加的按钮之后，观察到组件的渲染情况：

![](react性能优化/1554619384509.jpg)

我们发现按钮通过回调函数改变了父组件state中的count值，触发了父组件的渲染，同时Count组件依赖与父组件的state.count，所以也发生渲染。但是最奇怪的是为什么我的两个按钮都要去重新触发渲染函数呢？这里按钮的渲染是完全没有必要的，如何优化呢？

## shouldComponentUpdate 生命周期

`react` 提供了一个 `shouldComponentUpdate` 就是专门来解决这个问题的。这个生命周期函数就是决定这个组件是否要去渲染，在默认的情况下都是返回的 `true`，都要去重新渲染。这个函数接收两个参数 `nextProps`，`nextState` 方便调用者去判断这个组件是否需要渲染。

这里我们改造一下 `Button` 组件，让他不去做没必要的渲染。

```jsx harmony
class Button extends React.Component {
    // xxx省略
    shouldComponentUpdate(nextProps) {
        const { onClick, text } = this.props
        return nextProps.onClick !== onClick || nextProps.text !== text
    }
    render() {
        console.log('count button render')
        // xxx省略
    }
}
```

结果如下：

![](react性能优化/1554620664070.jpg)

button在触发了父组件的渲染后就不再重复渲染按钮组件了, 这是因为传给button的props.text和props.onClick没有发生改变。

我们似乎很容易的解决了这个问题。但是如果稍微修改下 `Counter` 组件给 `Button` 的传参方式如下：

```jsx harmony
class Counter extends React.Component {
    state = {
        count: 0
    }
    increase = () => {
        this.setState(state => ({
            count: state.count + 1
        }))
    }
    decrease = () => {
        this.setState(state => ({
            count: state.count - 1
        }))
    }
    render() {
        const { count } = this.state
        console.log('count container page render')
        return (
            <div className={styles.home}>
                <Count count={count} />
                <div>
                    {/* 修改这里传参的方式，注意比较上面的传参方式的区别 */}
                    <Button onClick={() => this.decrease()} text="-" />
                    <Button onClick={() => this.increase()} text="+" />
                </div>
            </div>
        )
    }
}
```

结果如下：

![](react性能优化/1554619384509.jpg)

哦嚯完蛋，这又回到之前没优化前的状态。这是什么原因导致的呢？这里Counter组件每次渲染的时候都是重新生成一个新的函数（虽然重新生成的这个函数功能和之前的函数功能是一摸一样的）,函数本质就是对象，是一种应用类型的数据，并非是基本类型数据，所以在Button内部的shouldComponentUpdate中比较的时候，尽管这两个函数功能是一样的，但是两个对象比较的是引用地址，所以会让Button组件以为props发生了改变，所以会重新渲染。

作为 `Button` 组件来说，即使内部针对渲染做了优化，但是由于外层组件调用者的传参不当同样会造成性能优化失效。

一点衍生：注意到有人喜欢在传参的时候直接去对方法去bind(this)，比如这样：
```jsx harmony
<Button onClick={this.increase.bind(this)} text="-" />
```
这样也同样也会造成相同的问题。

### React.PureComponent

这是一个官方推出来默认自带 `shouldComponentUpdate` 浅比较的组件，和 `React.Component` 功能相同，可以帮助我们不用手动去写 `shouldComponentUpdate`。但是值得注意的是，PureComponent实现的 `shouldComponentUpdate` 是浅比较的。

```jsx harmony
class Button extends React.PureComponent {
    static propTypes = {
        text: PropTypes.string,
        onClick: PropTypes.func
    }

    static defaultProps = {
        text: '',
        onClick() {}
    }

    render() {
        console.log('count button render')
        const { text, onClick: onClickProps } = this.props
        return (
            <button type="button" onClick={onClickProps}>
                {text}
            </button>
        )
    }
}
```

和上面的 `shouldComponentUpdate` 效果是一样的。

## functional component

我们观察发现，实现渲染优化的方式是使用 `class component` 中提供的 `shouldComponentUpdata` 方法，但是 functional component 相对于 class component 而言，更加的纯粹、轻量化、简洁明了，只有props去控制组件的渲染逻辑，那么functional component 能否做到渲染优化呢？

> react v16.6 带来了 React.memo

在低于16.6版本一下的react版本中，functional component 实际上是不太能优化渲染方式的，新版本中可以用 React.memo 包裹 functional component 实现渲染优化。

```jsx harmony
function Button ({text, onClick}) {
    console.log('count button render')
    return (
        <button type='button' onClick={onClick}>
            {text}
        </button>
    )
}
const MemoButton = React.memo(Button)
```

## todoList 另一个例子

从上面看到引用类型在比较的时候是存在一些问题的，这里看下面这个todoList的例子，来更深入理解下：

```jsx harmony
class TodoList extends Component {
    static index = 0
    state = {
        todos: [],
        inputValue: ''
    }
    addTodoListItem = () => {
        const { inputValue } = this.state
        if (inputValue) {
            this.setState(state => {
                state.todos.push({
                    isComplete: false,
                    id: TodoList.index++,
                    info: {
                        text: inputValue,
                        date: DateFunction.currentTime
                    }
                })
                return {
                    inputValue: '',
                    todos: state.todos
                }
            })
        }
    }
    deleteTodoListItem = ({ id }) => {
        this.setState(state => {
            const index = BaseFunction.getIndexFromListById(state.todos, id)
            BaseFunction.removeItemFromListByIndex(state.todos, index)
            return {
                todos: state.todos
            }
        })
    }
    handleComplete = ({ id }) => {
        this.setState(state => {
            const index = BaseFunction.getIndexFromListById(state.todos, id)
            const todo = state.todos[index]
            state.todos[index] = {
                ...todo,
                isComplete: !todo.isComplete
            }
            return {
                todos: state.todos
            }
        })
    }
    handleInputValueChange = e => {
        this.setState({
            inputValue: e.target.value
        })
    }
    render() {
        console.log('todoList is render')
        const { todos, inputValue } = this.state
        return (
            <div>
                <TodoInput
                    value={inputValue}
                    onChange={this.handleInputValueChange}
                    onComplete={this.addTodoListItem}
                    placeholder="请输入待办项"
                />
                {!todos.length && <span>暂无数据</span>}
                {todos.map(todo => (
                    <TodoListItem
                        handleDelete={this.deleteTodoListItem}
                        handleComplete={this.handleComplete}
                        data={todo}
                        key={todo.id}
                    />
                ))}
            </div>
        )
    }
}
```

