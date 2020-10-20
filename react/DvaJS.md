# DvaJS

- React 本身只是一个 DOM 的抽象层，使用组件构建虚拟 DOM

- 数据流

  - Flux，单向数据流方案，以 [Redux](https://github.com/reactjs/redux) 为代表
  - Reactive，响应式数据流方案，以 [Mobx](https://github.com/mobxjs/mobx) 为代表
  - 其他，比如 rxjs 等

- 流行的架构

  - 路由： [React-Router](https://github.com/ReactTraining/react-router/tree/v2.8.1)
  - 架构： [Redux](https://github.com/reactjs/redux)
  - 异步操作： [Redux-saga](https://github.com/yelouafi/redux-saga)

- dva 是React 应用框架,dva = React-Router + Redux + Redux-saga

- dva数据流

  数据的改变发生通常是通过用户交互行为或者浏览器行为（如路由跳转等）触发的，当此类行为会改变数据的时候可以通过 `dispatch` 发起一个 action，如果是同步行为会直接通过 `Reducers` 改变 `State` ，如果是异步行为（副作用）会先触发 `Effects` 然后流向 `Reducers` 最终改变 `State

  ![img](https://zos.alipayobjects.com/rmsportal/hUFIivoOFjVmwNXjjfPE.png)

  ![img](https://zos.alipayobjects.com/rmsportal/PPrerEAKbIoDZYr.png)

  ![img](https://zos.alipayobjects.com/rmsportal/pHTYrKJxQHPyJGAYOzMu.png)

  

  - State：一个对象，保存整个应用状态

    储存数据的地方，收到 Action 以后，会更新数据
  
  - View：React 组件构成的视图层. 
  
    React 组件构成的 UI 层，从 State 取数据后，渲染成 HTML 代码。只要 State 有变化，View 就会自动更新
  
- Action：一个对象，描述事件
  
  一个普通 javascript 对象，它是改变 State 的唯一途径。无论是从 UI 事件、网络回调，还是 WebSocket 等数据源所获得的数据，最终都会通过 dispatch 函数调用一个 action，从而改变对应的数据。action 必须带有 `type` 属性指明具体的行为，其它字段可以自定义，如果要发起一个 action 需要使用 `dispatch` 函数；需要注意的是 `dispatch` 是在组件 connect Models以后，通过 props 传入的
  
    ```js
    {
      type: 'click-submit-button',
      payload: this.form.data
    }
    ```
  
- connect 方法：一个函数，绑定 State 到 View
  
  ```js
    import { connect } from 'dva';
  
    function mapStateToProps(state) {
      return { todos: state.todos };
    }
    connect(mapStateToProps)(App);
  ```
  
- dispatch 方法：一个函数，发送 Action 到 State
  
    dispatching function 是一个用于触发 action 的函数，action 是改变 State 的唯一途径，但是它只描述了一个行为，而 dipatch 可以看作是触发这个行为的方式，而 Reducer 则是描述如何改变数据的
  
    被 connect 的 Component 会自动在 props 中拥有 dispatch 方法
    
    ```js
    dispatch({
      type: 'click-submit-button',
      payload: this.form.data
  })
  ```
  
- dva 提供 `app.model` 这个对象，所有的应用逻辑都定义在它上面

  - namespace: 当前 Model 的名称。整个应用的 State，由多个小的 Model 的 State 以 namespace 为 key 合成

  - state: 该 Model 当前的状态。数据保存在这里，直接决定了视图层的输出

  - reducers: Action 处理器，处理同步动作，用来算出最新的 State

  - effects：Action 处理器，处理异步动作

    Effect 是一个 Generator 函数，内部使用 yield 关键字，标识每一步的操作（不管是异步或同步）,  之所以叫副作用是因为它使得我们的函数变得不纯，同样的输入不一定获得同样的输出

    - call：执行异步函数
    - put：发出一个 Action，类似于 dispatch

  - subscription

    是一种从 **源** 获取数据的方法，Subscription 语义是订阅，用于订阅一个数据源，然后根据条件 dispatch 需要的 action

  ```js
  const app = dva();
  
  app.model({
    namespace: 'count',
    state: 0,
    reducers: {
      add(state) { return state + 1 },
    },
    effects: {
      *addAfter1Second(action, { call, put }) {
        yield call(delay, 1000);
        yield put({ type: 'add' });
      },
    },
  });
  
  app.router(() => <App />);
  app.start('#root');
  ```

  