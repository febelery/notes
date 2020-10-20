# Redux

## 设计思想

- Web 应用是一个状态机，视图与状态是一一对应的。
- 所有的状态，保存在一个对象里面。



## 基本概念和 API

### Store

`Store` 就是保存数据的地方，你可以把它看成一个容器。整个应用只能有一个 Store

### State

`Store`对象包含所有数据。如果想得到某个时点的数据，就要对 Store 生成快照。这种时点的数据集合，就叫做 `State`

Redux 规定， 一个 State 对应一个 View。只要 State 相同，View 就相同

### Action

State 的变化，会导致 View 的变化。但是，用户接触不到 State，只能接触到 View。所以，State 的变化必须是 View 导致的。Action 就是 View 发出的通知，表示 State 应该要发生变化了。

Action 是一个对象。其中的`type`属性是必须的，表示 Action 的名称

Action 描述当前发生的事情。改变 State 的唯一办法，就是使用 Action。它会运送数据到 Store

### Action Creator

View 要发送多少种消息，就会有多少种 Action。如果都手写，会很麻烦。可以定义一个函数来生成 Action，这个函数就叫 Action Creator

### Dispatch

`store.dispatch()`是 View 发出 Action 的唯一方法

### Reducer

Store 收到 Action 以后，必须给出一个新的 State，这样 View 才会发生变化。这种 State 的计算过程就叫做 Reducer。

Reducer 是一个函数，它接受 Action 和当前 State 作为参数，返回一个新的 State

Reducer 函数最重要的特征是，它是一个纯函数。也就是说，只要是同样的输入，必定得到同样的输出

#### 纯函数

- 不得改写参数
- 不能调用系统 I/O 的API
- 不能调用`Date.now()`或者`Math.random()`等不纯的方法，因为每次会得到不一样的结果

Redux 提供了一个`combineReducers`方法，用于 Reducer 的拆分

### Subscribe

Store 允许使用`store.subscribe`方法设置监听函数，一旦 State 发生变化，就自动执行这个函数

只要把 View 的更新函数（对于 React 项目，就是组件的`render`方法或`setState`方法）放入`listen`，就会实现 View 的自动渲染

`store.subscribe`方法返回一个函数，调用这个函数就可以解除监听