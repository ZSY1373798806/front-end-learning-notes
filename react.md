### React核心设计思想

- #### 组件化设计

  - 原子化构建 UI分解为独立的功能单元（如函数式组件/类组件）

  - 组合模式 通过props嵌套实现组件树结构

  - 隔离型 每个组件维护自身状态和样式（CSS-in-JS生态支持）

  - 复用机制 高阶组件（HOC）与自定义Hooks实现逻辑复用

- #### 声明式编程范式

  - 状态驱动 UI=f(state)的数学表达式（无需手动操作DOM）

  - 抽象渲染 开发者专注描述目标状态，框架处理渲染细节

  - 幂等性保证 相同的state必定输出相同视图（确定性原则）

- #### 高效更新引擎

  - 虚拟DOM层 内容中轻量级DOM表示（对象树结构）

  - Diff算法优化 基于树遍历策略

  - 批量更新策略 自动合并setState操作（异步更新策略）

  - 渲染流水线 Fiber结构实现可中断渲染（并发模式）

- #### 单向数据控制

  - 严格数据通道 props自上而下传递（严禁子级逆向修改）
  - 状态托管 通过Context/Redux实现跨层级通信
  - 副作用隔离 Hooks机制约束副作用边界（useEffect依赖链）

### useEffect 工作原理

useEffect钩子的工作原理涉及到React的渲染流程和副作用的调度机制

- #### 调度副作用

  - 当在组件内部副调用useEffect时，实际上是将一个副作用函数及其依赖项数组排队等待执行；这个函数不会立即执行

- #### 提交阶段

  - React渲染组件并执行了所有纯函数组件或类组件的渲染方法后，会进入提交阶段
  - 在这个阶段，将计算出的新视图（新的DOM节点）更新到屏幕上
  - 一旦更新完成，React就知道现在可以安全的执行副作用函数了，因为这不会影响到正在屏幕上显示的界面

- #### 副作用执行

  - 提交阶段完成后，React会处理所有排队的副作用
  - 如果组件是首次渲染，所有副作用都会执行
  - 如果组件是重新渲染，React会首先对比副作用的依赖数组：如果依赖项未变，副作用不会执行；如果依赖性变化 或者没有提供依赖，则会再次执行

- #### 清理机制

  - 如果副作用函数返回一个函数，那么这个函数将视为清理函数
  - 在执行当前的副作用之前，以及组件卸载前，React会先调用上一次渲染中的清理函数；这样确保副作用不会内存泄漏，同时能撤销上一次副作用导致的改变

通过这种机制，`useEffect` 允许开发者以一种优化的方式来处理组件中可能存在的副作用，而不需要关心渲染的具体时机。退出清理功能确保了即使组件被多次快速创建和销毁，应用程序也能保持稳定和性能。

- #### 延迟副作用

  - 尽管useEffect会在渲染之后执行，但它是异步执行的，不会阻塞浏览器更新屏幕
  - 这意味着React会等浏览器完成绘制之后，在执行副作用函数，以此确保副作用处理不会导致用户可见的延迟


### useEffect & useLayoutEffect

- #### useLayoutEffect

  - useLayoutEffect 是同步执行的钩子函数，在 DOM 更新后立即执行，可能会阻塞浏览器的绘制过程

- #### useEffect 

  - useEffect 是异步执行的钩子函数，在浏览器完成绘制后延迟执行，不会阻塞浏览器的绘制过程


### 性能优化

- #### useCallback

  用于函数的缓存

  等价于 useMemo(() => () => {}, []);

-  #### useMemo

  用户函数结果的缓存

- #### React.memo

  用于组件记缓存，避免不必要的重新渲染(浅比较props)

### 合成事件

- #### 合成事件的优势

  - 抹平不同浏览器直接的差异，提供统一的API使用体验
  - 通过事件委托的方式，统一绑定和分发事件，有利于提升性能，减少内存消耗

- #### 合成事件的绑定及分发流程

  - react应用在启动时，会在页面渲染的根元素（<div id="root"></div>）上绑定原生的DOM事件，将该跟元素作为委托对象
  - 在组件渲染时，会通过JSX解析出元素上绑定的事件，并将这些事件与原生事件进行一一映射
  - 当用户点击页面元素时，事件会冒泡到根元素，之后根元素监听的事件通过dispatchEvent方法进行事件派发
  - dispatchEvent会根据事件的映射关系以及DOM元素，找到react中与之对应的fiber节点
  - 找到fiber节点之后，将其绑定的合成事件函数加到一个函数执行队列中
  - 最后依次执行队列中的函数，完成事件的触发流程

### 实现一个useState Hook

```typescript
import { useRef, useReducer } from 'react';

const useState = defaultValue => {
    const value = useRef(defaultValue);

    const [, dispatch] = useReducer(() => ({}), {});

    const setValue = newValue => {
        if (typeof newValue === 'function') {
            value.current = newValue(value.current);
        } else {
            value.current = newValue;
        }
        dispatch();
    }

    return [value.current, setValue];
}
```

### react 架构

- #### Reconciler（调度器）

  - 调度任务的优先级，高于任务优先进入Reconciler
  - 为什么使用requestIdleCallback
    - 浏览器兼容性
    - 触发频率不稳定，收很多因素影响（比如浏览器切换tab时，之前tab注册的requestIdleCallback触发频率会变得很低）

- #### Scheduler（协调器）

  - 负责找出变化的组件

- #### Renderer（渲染器）

  - 负责将变化的组件渲染到页面上


### react的patch流程

- react有一个Scheduler调度器，主要用于调度fiber节点的生成和更新任务
- 当组件更新时，Reconciler协调器执行组件的render方法，生成一个fiber节点之后，再递归的去生成fiber节点的子节点
- 每个fiber的生成都是一个单独的任务，会医回调的形式交给Scheduler进行调度处理，在Scheduler里会根据任务的优先级去执行任务
- 任务的优先级的指定是根据【车道模型】，将任务进行分类，每一类拥有不同的优先级，所有的分类和优先级都在react中进行了枚举
- Scheduler按照优先级执行任务时，会异步的执行，同时没一个任务执行完成之后，都会通过requestIdleCallback去判断下一个任务是否能在当前渲染的剩余时间内完成
- 如果不能完成就发生中断，把线程的控制权交给浏览器，剩下的任务则在下一个渲染帧内执行
- 整个Reconciler和Scheduler的任务执行完成之后，会生成一个新的workInProgressFiber节点数，之后Reconciler触发Commit阶段通知Render渲染器去进行diff操作，就是所谓的patch流程

### react的diff算法

- #### 单节点diff

  - 首先会判断老的Fiber树上有没有对应的Fiber节点，若没有则说明是新增操作，直接在老Fiber树上新增节点并更新DOM
  - 若老Fiber节点也存在，则判断节点上的`key`值是否相同，若不同则删除老节点并新增新节点
  - 若`key`值相同，则判断节点的`type`是否相同，若不同则删除老节点并新增节点
  - 若`type`值也相同，则认为是一个可复用的节点，直接返回老节点就行

- #### 多节点diff

  多节点的Diff操作主要用于map返回多个相同节点的情况下，可以分为三种情况：新增节点、删除节点以及节点移动，React采用双重遍历的方式来进行三种情况的判断，流程如下：

  - 第一轮遍历会依次将 children[i] 和 currentFiber 以及 children[i++] 和 currentFiber.sibling 进行对比，当发现节点不可复用时提前结束遍历
  - 当第一轮遍历无提前结束时，说明所有节点都可以复用，直接返回老节点
  - 若children遍历完成，currentFiber未完成，则说明是删除操作，需要对未完成的 currentFiber 兄弟节点标记删除
  - 若children遍历未完成，currentFiber完成，则说明是新增操作，需要生成新的workInProgressFiber节点
  - 若children和currentFiber都未完成，则说明是节点位置发送了变更，那就对剩余的currentFiber进行遍历，并通过key值找到每一个节点在children中对应的老节点，并将老节点中的位置替换为新节点的

### setState之后，发生了哪些事情

setState就是触发组件的一次渲染过程

- setState 会生成一份新的组件内状态数据，并重新执行Reconciler中的render方法
- render方法会根据JSX和最新的数据去创建一个新的fiber节点数，没一个节点数的创建都是Reconciler中的一个工作单元
- 所有的创建fiber节点工作生成后，这些工作单元的执行和调度会由Scheduler中的任务队列来执行
- 任务队列每次取出一个创建fiber节点的任务执行，执行完成之后会调用浏览器的requestIdleCallback方法来判断当前刷新帧剩余时间是否够执行下一个任务
- 如果时间够，就执行下一个创建fiber节点任务，不够的话就先将创建任务暂停，等下一个刷新帧继续执行
- 当所有的创建任务都执行完成之后，就生成了一颗新的fiber节点数，之后就是通过新旧两棵树去做diff算法获得要更新的树，后面就是diff和渲染的过程

### Hook核心原理

- #### 闭包与链表存储

  - 存储结构

    Hooks数据存储在Fiber节点的memoizedState属性中，通过单向链表管理

  - 执行顺序依赖

    Hooks调用顺序在每次渲染中必须严格一致（链表顺序不可变）

  - 闭包陷阱

    每个Hooks闭包捕获单次渲染的props/state快照

```js
// Fiber 节点结构示意
const fiber = {
  memoizedState: {
    memoizedState: '状态值',    // useState 的状态
    next: {                   // 下一个 Hook
      memoizedState: [],      // useEffect 的依赖数组
      next: null
    }
  }
};
```

- #### 调度机制

  - 优先级调度

    Hooks更新请求会被Scheduler模块根据优先级（Immediate/UserBlocking/Normal）排队处理

  - 批量更新

    React自动合并多个setState调用，减少渲染次数

### Fiber

#### 背景

- ##### 旧版协调算法瓶颈

  - 递归不可中断

    同步遍历整个虚拟DOM树，长时间占用主线程

  - 卡顿问题

    复杂组件树更新导致掉帧（如大型列表，动画场景）

- ##### 目标

  - 实现增量渲染，支持异步可中断的更新

#### 核心设计思想

- ##### 时间切片

  - 将渲染任务拆分为多个小任务（Fiber节点）
  - 利用浏览器空闲时段（requestIdleCallback）分片执行

- ##### 优先级调度

  - 用户交互（如输入）优先于数据更新（如API响应）

- ##### 可恢复工作单元

  - 保存中间状态，允许暂停/恢复渲染流程

- ##### Fiber节点数据结构

  > 每个Fiber节点对应一个组件实例或DOM节点，包含一下核心属性

| 属性          | 类型          | 作用                                         |
| ------------- | ------------- | -------------------------------------------- |
| type          | String/Object | 组件类型（如 'div'、函数组件引用）           |
| stateNode     | Object        | 对应的真实 DOM 节点或类组件实例              |
| child         | Fiber         | 第一个子节点                                 |
| sibling       | Fiber         | 下一个兄弟节点                               |
| return        | Fiber         | 父节点                                       |
| pendingProps  | Object        | 新传入的 props                               |
| memoizedProps | Object        | 上一次渲染使用的 props                       |
| memoizedState | Object        | 上一次渲染后的 state（如 Hooks 链表）        |
| effectTag     | Number        | 标记副作用（如 Placement、Update、Deletion） |
| alternate     | Fiber         | 指向当前 Fiber 的镜像（用于 Diff 比较）      |

- ##### Fiber双缓冲机制

  > React维护两颗Fiber树：保证渲染过程中Current Tree始终完整可用，避免更新过程中出现UI不一致

  - Current Tree：当前已渲染的UI对应的Fiber树
  - WorkInProgress Tree：正在构建的新Fiber树

  > 切换流程：更新完成后，workInProgress Tree树变成Current树

  ```text
  初始渲染：
  Current Tree: null
  WorkInProgress Tree: → 构建完成 → 提交后成为 Current Tree
  
  更新阶段：
  Current Tree ←→ WorkInProgress Tree（复用或新建节点） 
  ```

  > 两颗树通过alternate属性连接

  ```js
  currentFiber.alternate === workInProgressFiber;
  workInProgressFiber.alternate === currentFiber;
  ```

#### Fiber渲染

- ##### 协调阶段

  - 目标

    - 生成副作用列表，不修改DOM；负责计算变更

  - 过程

    - 递阶段：调用render生成子节点，标记变化

      - beginWork【packages\react-reconciler\src\ReactFiberBeginWork.old.js】

        ```typescript
        
        /**
         * current: 当前组件对应的Fiber节点在上一次更新时的Fiber节点，即workInProgress.alternate
         * workInProgress: 当前组件对应的Fiber节点
         * renderLanes: 优先级相关
         */
        function beginWork(current: Fiber | null, workInProgress: Fiber, renderLanes: Lanes): Fiber | null {
          // ...省略函数体
        }
        ```

        

      - 先从rootFiber开始向下深度优先遍历，为遍历到的每个Fiber节点调用beginWork方法

      - 该方法会根据传入的Fiber节点创建子Fiber节点，并将这两个Fiber节点连接起来

      - 当遍历到叶子节点（即没有子组件的组件）时就会进入【归】阶段

    - 归阶段：向上回溯收集副作用

      - completeWork【packages\react-reconciler\src\ReactFiberCompleteWork.old.js】
      - 在【归】阶段会调用completeWork处理Fiber节点
      - 当某个Fiber节点执行完completeWork，如果其存在兄弟Fiber节点（即fiber.sibling !== null），会进入其兄弟Fiber的【递】阶段
      - 如果不存在兄弟Fiber，会进入父级Fiber的【归】阶段
      - 【递】和【归】阶段会交错执行，直到【归】到rootFiber；至此，render阶段的工作就结束了

  - 可中断

    - 根据剩余时间片暂停/恢复遍历

- ##### 提交阶段

  - 目标
    - 同步执行所有DOM变更；执行DOM操作
  - 过程
    - Before Mutation：调用getSnapshotBeforeUpdate
    - Mutation：执行DOM增删改
    - Layout：调用useLayoutEffect和componentDidMount/Update
  - 不可中断
    - 避免中间状态导致UI不一致

- ##### 整个更新流程

  其中红框部分的步骤随时可能由于一下原因被中断

  - 有其他更高优任务需要先更新
  - 当前帧没有剩余时间

> 由于红框中的工作都在内存中进行，不会更新页面上的DOM，所以即使反复中断，用户也不会看见更新不完全的DOM

<img src="https://react.iamkasong.com/img/v15.png" alt="更新流程" style="zoom: 33%;" />

<img src="https://react.iamkasong.com/img/process.png" alt="更新流程" style="zoom:33%;" />

```react
function App() {
  return (
    <div>
      i am
      <span>KaSong</span>
    </div>
  )
}
```

对应Fiber树结构

<img src="https://react.iamkasong.com/img/fiber.png" alt="Fiber架构" style="zoom: 25%;" />

render阶段会依次执行

```js
1. rootFiber beginWork
2. App Fiber beginWork
3. div Fiber beginWork
4. "i am" Fiber beginWork
5. "i am" Fiber completeWork
6. span Fiber beginWork
7. span Fiber completeWork
8. div Fiber completeWork
9. App Fiber completeWork
10. rootFiber completeWork
```

> 之所以没有 “KaSong” Fiber 的 beginWork/completeWork，是因为作为一种性能优化手段，针对只有单一文本子节点的`Fiber`，`React`会特殊处理。

#### 优先级调度模型

> 调度策略：高优先级任务可中断低优先级任务

| 优先级               | 对应场景                    |
| -------------------- | --------------------------- |
| NoPriority           | 无优先级                    |
| ImmediatePriority    | 同步任务（如 flushSync）    |
| UserBlockingPriority | 用户交互（点击、输入）      |
| NormalPriority       | 数据更新、网络响应          |
| LowPriority          | 过渡更新（Concurrent 模式） |
| IdlePriority         | 空闲时执行的任务            |

#### Fiber对生命周期的影响

- ##### 废弃生命周期

  - `componentWillMount、componentWillReceiveProps、componentWillUpdate`
  - 原因：异步可渲染可能导致多次调用，引发副作用错误

- ##### 新增API

  - getDerivedStateFromProps（静态方法，代替componentWillReceiveProps）
  - getSnapshotBeforeUpdate（代替componentWillUpdate）

#### Fiber与并发模式

- ##### 并发特性

  - useTranstion：标记非紧急更新，可被高优先级任务打断
  - Suspense：等待异步数据加载时显示回退UI

- ##### 启用方式

```js
// 创建根节点时启用
ReactDOM.createRoot(document.getElementById('root')).render(<App />);
```

### React源码的文件结构

```bash
react/
├── packages/  # 包含元数据（比如 package.json）和 React 仓库中所有 package 的源码（子目录 src）
│   ├── create-subscription/
│   ├── dom-event-testing-library/
│   ├── eslint-plugin-react-hooks/
│   ├── jest-react/
│   ├── react/ # React的核心，包含所有全局 React API，如：React.createElement，React.Component，React.Children
│   ├── react-art/ # Renderer相关的文件夹
│   ├── react-cache/
│   ├── react-client/  # 创建自定义的流
│   ├── react-debug-tools/
│   ├── react-devtools/
│   ├── react-devtools-core/
│   ├── react-devtools-extensions/
│   ├── react-devtools-inline/
│   ├── react-devtools-shared/
│   ├── react-devtools-shell/
│   ├── react-dom/ # Renderer相关的文件夹；同时是DOM和SSR（服务端渲染）的入口
│   ├── react-dom-bindings/
│   ├── react-fetch/ # 用于数据请求
│   ├── react-flight/
│   ├── react-interactions/ # 用于测试交互相关的内部特性，比如React的事件模型
│   ├── react-is/ # 用于测试组件是否是某类型
│   ├── react-native-renderer/ # Renderer相关的文件夹
│   ├── react-noop-renderer/ # Renderer相关的文件；用于debug fiber
│   ├── react-reconciler/ # Reconciler的实现，你可以用他构建自己的Renderer；对接Scheduler和不同平台的Renderer，构成了整个 React1的架构体系
│   ├── react-refresh/ # “热重载”的React官方实现
│   ├── react-server/ # 创建自定义SSR流
│   ├── react-server-dom-webpack/
│   ├── react-test-renderer/
│   ├── scheduler/ # Scheduler（调度器）的实现
│   ├── shared/ # 源码中其他模块公用的方法和全局变量，比如在shared/ReactSymbols.js中保存React不同组件类型的定义
│   └── use-subscription/
├── scripts/  # 各种工具链的脚本，比如git、jest、eslint等
│   ├── build/
│   ├── rollup/
│   └── tasks/
├── fixtures/ # 包含一些给贡献者准备的小型 React 测试项目
│   ├── dom/
│   ├── reconciler/
│   └── devtools/
└── build/
```

