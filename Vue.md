## Vue
### 生命周期
- create阶段：vue实例被创建
	- beforeCreate：创建前，data、methods中的数据还没初始化
	- created：创建完，data中有值，未挂载
- mount阶段：vue实例被挂载到真实dom节点
	- beforeMount：可以发起服务请求，取数据
	- mounted：可以操作dom
- update阶段：当vue实例里面的data数据变化时，触发组件的重新渲染
	- beforeUpdate：更新前
	- updated：更新后
- destroy阶段：vue实例被销毁
	- beforeDestroy：实例被销毁前，此时可以手动销毁一些方法
	- destroyed： 销毁后
- 执行顺序
	- 组件创建
		父beforeCreate => 父create => 父beforeMount => 子beforeCreate => 子created => 子beforeMounted => 父mounted
	- 子组件更新
		父beforeUpdate => 子beforeUpdate => 子updated => 父updated
	- 父组件更新
		父beforeDestroy => 父updated
	- 父组件销毁
		父beforeDestroy => 子beforeDestroy => 子destroyed => 父destroyed
	
	![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220211225838.png)
### Vue.use
	引入一个插件，主要是plugin.install实现的
	```js
	if (typeof plugin.install === 'function') {
	  plugin.install.apply(plugin, args);
	} else if (typeof plugin === 'function') {
	  plugin.apply(null, args);
	}
	```
- 原理
- 实现
### nextTick

- #### 作用

  - 解决异步更新问题：确保能获取更新后的DOM
  - 在下次DOM更新循环结束之后执行延迟回调，用于在数据变化后立即操作更新后的DOM

- #### 为什么无法直接获取更新后的DOM

  - JavaScript单线程特性
    - 同步代码必须全部执行完，才会处理异步任务
  - Vue的异步更新策略
    - 为了性能优化，DOM更新被延迟到下一个事件循环
  - 事件循环时机
    - 数据修改是同步的，DOM更新时异步的

- #### 原理

  - 利用事件循环
    - Vue的nextTick实现利用了JavaScript的事件循环机制；在浏览器环境中，JavaScript是单线程执行的，事件循环负责管理异步任务的执行顺序
    - Vue将nextTick的回调函数放入微任务或宏任务队列
    - 当当前执行栈为空时，事件循环会从任务队列中取出任务执行；如果微任务队列中有任务，会先执行微任务队列中的任务，然后在执行宏任务队列中的任务；这样可以确保nextTick的回调函数在DOM更新之后执行
  - 内部实现
    - Vue内部维护了一个异步任务队列，用于存储nextTick的回调函数；当调用nextTick时，回调函数会被添加到这个队列中
    - Vue在更新DOM之后，会检查这个异步任务队列是否为空；如果不为空，会取出队列中的第一个任务并执行
    - 这样就保证了在DOM更新完成之后，nextTick的回调函数能够按照调用的顺序依次执行

  ![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220219161533.png)

- #### 为什么优先是微任务
	主要使用了宏任务和微任务；
	根据执行环境分别尝试用Promise、MutationObserver、setImmediate；
	如果以上都不行，则采用setTimeout定义一个异步方法；
	多次调用nextTick会将方法存入队列中，通过这个异步方法情况执行当前队列；
	
	- 为什么优先使用Promise
	
	  - 核心原因：**性能和兼容性的平衡**。
	
	    1. **性能最优**：
	
	       - `Promise` 创建微任务的性能远高于 `MutationObserver` 或 `setImmediate`。
	
	         - 为什么
	
	           | **层级**       | `Promise`                | `MutationObserver`                    | `setImmediate`             |
	           | :------------- | :----------------------- | :------------------------------------ | :------------------------- |
	           | **执行环境**   | JS 引擎微任务队列        | 渲染引擎回调队列                      | 事件循环宏任务队列         |
	           | **调度优先级** | 最高（当前任务末尾）     | 高（渲染前）                          | 低（渲染后）               |
	           | **跨线程通信** | 无                       | 需要 JS ↔ 渲染引擎                    | 需要                       |
	           | **创建开销**   | ⚡️ 极低（纯 JS 引擎操作） | 🚧 高（需创建 DOM 节点+监听+触发修改） | 🚧 中（跨层调用浏览器 API） |
	           | **创建开销**   | ⚡️ 直接进入微任务队列     | 🚧 需要浏览器渲染引擎介入              | 🚧 宏任务队列调度           |
	
	       - 现代浏览器普遍支持 `Promise`，执行效率最高。
	
	    2. **执行时机更早**：
	
	       - 微任务中，`Promise` 比 `MutationObserver` 触发更早（部分浏览器中 `MutationObserver` 优先级略低）。
	
	    3. **降级策略**：
	
	       - 当 `Promise` 不可用时（如 IE11），Vue 会按优先级降级：
	         1. `MutationObserver`（监听 DOM 变动触发微任务）
	         2. `setImmediate`（IE 专属，比 `setTimeout` 更快）
	         3. `setTimeout`（宏任务，最后兜底）
	
- #### 流程
	
	- 把回调函数放入callbacks等待执行
	- 将执行的函数放到微任务或宏任务中
	- 事件循环到了微任务或宏任务，执行函数依次执行callbacks中的回调
> DOM更新是同步的；UI渲染是在所有微任务完成之后，是异步的；我们在nextTick里拿到的数据，是更新后的DOM，因为diff算法和patch补丁已经算出来了，并作用在DOM上

```js
// Vue 源码节选（next-tick.js）
if (typeof Promise !== 'undefined') {
  // 优先使用 Promise
  const p = Promise.resolve()
  timerFunc = () => p.then(flushCallbacks)
} else if (typeof MutationObserver !== 'undefined') {
  // 降级方案 1：MutationObserver
  const observer = new MutationObserver(flushCallbacks)
  // 监听一个文本节点，通过修改内容触发回调
} else if (typeof setImmediate !== 'undefined') {
  // 降级方案 2：setImmediate（IE）
  timerFunc = () => setImmediate(flushCallbacks)
} else {
  // 最终降级：setTimeout
  timerFunc = () => setTimeout(flushCallbacks, 0)
}

/**
 * 做了3件事
 * 1、将pending置为false
 * 2、清空callbacks数组
 * 3、执行callbacks数组中的每一个函数（比如flushSchedulerQueue、用户调用nextTick传递的回调函数）
 */
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  /* 遍历callbacks数组，执行其中存储的每个flushSchedulerQueue函数 */
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

/**
 * 完成两件事
 * 1、用try catch包装flushSchedulerQueue函数，然后将其放入callbacks数组
 * 2、如果pending为false，表示现在浏览器的任务队列中没有flushCallbacks函数；
 *  如果pending为true，则表示浏览器的任务队列中已经被放入了flushCallbacks函数；
 *  待执行flushCallbacks函数时，pending会被再次置为false，表示下一个flushCallbacks函数可以进入浏览器的任务队列了
 * pending：保证在同一时刻，浏览器的任务队列只有一个flushCallbacks函数
 * @param {*} cb 接收一个回调函数 => flushSchedulerQueue
 * @param {*} ctx 上下文
 * @returns 
 */
/**
 * 流程：
 *  1、把回调函数放入callbacks等待执行
 *  2、将执行的函数放到微任务或宏任务中
 *  3、事件循环到了微任务或宏任务，执行函数依次执行callbacks中的回调
 * 
 * DOM更新是同步的；
 * UI渲染是在所有微任务完成之后，是异步的；
 * 我们在nextTick里拿到的数据，是更新后的DOM，因为diff算法和patch补丁已经算出来的，并作用在DOM上
 */
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  /* 用callbacks数组存储经过包装的cb函数 */
  callbacks.push(() => {
    if (cb) {
      /* 用try catch包装回调函数，便于错误捕获 */
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    /* 执行timerFunc，在浏览器的任务队列中（首选微任务队列）放入flushCallbacks函数 */
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```



### 路由vue-router
- 钩子函数
	- 全局守卫：beforeEach，afterEach
	- 路由守卫：beforeEnter
	- 组件守卫：beforeRouteEnter，beforeRouteUpdate，beforeRouteLeave
- 路由模式
    - hash
    	hash 虽然出现在 URL 中，但不会被包括在 HTTP 请求中，对后端完全没有影响，因此改变 hash 不会重新加载页面
		- 原理
        	hashchange监听锚点
    - history
		- 原理
        	提供pushState和replaceState方法，这两个方法改变url的path不会引起页面刷新
        	提供了类似hashchange的popState事件：
        		通过浏览器前进后退改变url，会触发popState
        		通过浏览器pushState/replaceState或a标签改变url，不会触发popState；可通过拦截来检测url变化
        		通过js调用history的back、go、forward方法，触发popState
		- Nginx配置 try_files $uri $uri/ /index.html;
	- abstract
		用于node环境中

    - 实现
    	应用于浏览器的历史记录栈，在当前已有的 back、forward、go 的基础之上，它们提供了对历史记录进行修改的功能。只是当它们执行修改时，虽然改变了当前的 URL，但浏览器不会立即向后端发送请求。
- 路由懒加载
	为什么会更快 分包
- keep-alive
    - 作用
        实现组件缓存，保存这些组件的状态，避免反复渲染；
    - 生命周期：activated，deactivated
    	页面第一次进入，钩子的触发顺序created-> mounted-> activated；
    	退出时触发deactivated；
    	当再次进入（前进或者后退）时，只触发activated
    	
    - 原理
        vue内部将dom节点抽象成一个个VNode节点；keep-alive组件的缓存也是基于VNode节点；
        它将满足条件（pruneCache）的组件在cache对象中缓存起来；
        在需要重新渲染的时候再将VNode节点从cache对象中取出并渲染；
    - 踩坑
		- 多级路由嵌套，只缓存到二级，后面几层router-view中的组件缓存会出现问题
		解决：每次的router-view都包裹一层keep-alive
### Vue选项合并策略
### 双向绑定原理
	vue采用 数据劫持 结合 发布-订阅 模式；
- Observer监听器
	通过Object.defineProperty()来监听数据的变动，遍历对象，包括子对象，给每个属性加上getter，setter
	每当数据发生变化，就会触发setter；
	这时候Observer就要通知订阅者，订阅者就是Watcher；
- Compiler指令解析器
	解析模板指令；
	将模板中的变量替换成数据，然后初始化渲染页面视图；
	并将每个指令对应的节点绑定更新函数，添加监听数据的订阅者；
	一旦数据有变动，收到通知，调用更新函数更新视图；
- Watcher订阅者
	订阅者作为Observer和Compiler之间的通信桥梁，主要负责：
	- 在自身实例化时，往属性订阅器(dep)里添加自己；
	- 自身必须有一个update方法；
	- 当Observer属性变动，通知(dep.notice())时，能调用自身的update()方法，并触发Compile中的绑定回调
- Dep订阅器
	采用发布-订阅模式；
	用来收集订阅者Watcher；
	对监听器Observer和订阅者Watcher进行统一管理；
	![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220212103055.png)
```js
//创建订阅发布者
//1.管理订阅
//2.集体通知
class Dep {
  constructor() {
    this.subs = [];
  }
  //添加订阅
  addSub(sub) {//其实就是watcher对象
    this.subs.push(sub)
  }
  //集体通知
  notify() {
    this.subs.forEach((sub) => {
      sub.update()
    })
  }
}
```
### 数组响应式重写
- 实现
```js
const arrExtend = Object.create(Array.prototype)
// const arrExtend = {}
// arrExtend._proto_=Array.prototype
const arrMethods = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']
arrMethods.forEach(method => {
  const oldMethod = Array.prototype[method]
  const newMethod = function(...args) {
    oldMethod.apply(this, args)
    console.log(`${method} 方法被执行了`)
  }
  arrExtend[method] = newMethod
})
```


### 双向绑定的优点

### template原理和实现
![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220209123317.png)
### 虚拟dom
	数据改变-虚拟dom计算变更-操作真实dom-视图更新
- 格式：
	 {
		tag: 'div', // 选择器
		data: {class,attribute,style}, // 最后渲染成真实dom节点后，节点上的class、attribute、style及绑定的事件
		children: [{xxx}], // vnode子节点
		text: 'xxx', // 文本属性
		elm, //vnode对应的真实dom节点
		key, // vnode标记，提高diff的效率
	}
- 虚拟dom的好处
	虚拟dom是为了解决模板渲染问题；
	- 无需手动优化
	- 无需手动操作dom。只管代码逻辑
	- 会比较虚拟dom的变化，计算出最小的需要更新的视图，再去操作dom
- 为什么要用虚拟dom
- 与jQuery或原生操作dom的方式，有什么区别
- dom遍历方式
  - diff算法
- 实现原理
	- 用js对象模拟真实dom，对真实dom进行抽象；
	- diff算法，比较两颗dom数的差异；
	- patch算法，将差异更新应用到真实dom；
### diff算法
- 作用：
	diff算法用来比较两棵虚拟dom树的差异
	![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220214215959.png)
- 深度优先遍历，记录差异
	先子节点，后相邻节点（深度优先 递归updateChildren）
	![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220214215941.png)
- vdom的核心；
    - 只会比较同一层级；
    - 标签名不同，直接删除，不继续深度比较；
    - 标签名相同，key也相同，就认为是相同节点，不继续深度比较
- patch函数（vdom最核心的方法，完成视图更新的关键方法）
	- 作用
        完成oldVnode和vnode的diff过程；
        并根据需要操作的vdom节点打上patch；
        最后生成新的真实dom节点并完成视图更新工作；
	  -执行情况(两种)
		1. oldVnode不存在
			创建新节点
		2. oldVnode存在
			会对oldVnode和vnode进行diff及patch的过程；
			patch会调用sameVNode方法，来对两个传入的vnode进行基本属性比较；
			1. 只有当基本属性相同的情况，才认为这2个vnode只是局部发生更新；
			然后才会对2个vnode进行diff；
			2. 如果2个vnode基本属性存在不一致，会直接跳过diff过程；
			依据vnode新建一个真实的dom，同时删除老的dom节点；
	  	（oldCh=oldVnode的子节点，ch=vnode的子节点）
- patchVNode（diff过程的方法）
	- 进行文本节点判断，若oldVnode.text!==vnode.text，直接进行文本节点的替换；
	- vnode没有文本节点，进入子节点的diff；
	- oldCh和ch都存在且不同，调用updateChildren对子节点进行diff；
	- oldCh不存在，ch存在，清空oldVnode的文本节点，同时调用addVnodes方法，将ch添加到elm真实dom节点中；
	- oldCh存在，ch不存在，删除elm真实节点下的oldCh；
	- oldVnode有文本节点，而vnode没有，清空这个文本节点；
- updateChildren（子节点diff流程，整个diff过程中最重要的环节）
	- 首先给oldCh和ch分别分配一个startIndex和endIndex来作为遍历的索引；
	- 当oldCh或ch遍历完后（条件：oldCh或ch的startIndex>endIndex），停止oldCh和ch的diff过程；
    ![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220209204451.png)
### 初始化Vue到最终渲染的整个过程
![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220215214130.png)
### MVVM & MVC
### Proxy & Object.defineProperty
- Proxy
	- Proxy可以直接监听对象而非属性；
	- 可以直接监听数组的变化
	
	  ```js
	  const handler = {
	    get(target, prop) {
	      // 拦截所有属性访问
	      return Reflect.get(target, prop);
	    },
	    set(target, prop, value) {
	      // 拦截所有属性设置
	      return Reflect.set(target, prop, value);
	    },
	    deleteProperty(target, prop) {
	      // 拦截属性删除
	      return Reflect.deleteProperty(target, prop);
	    }
	  };
	  
	  const proxy = new Proxy([], handler);
	  ```
	
	  ```js
	  function reactive(obj) {
	    return new Proxy(obj, {
	      get(target, key) {
	        track(target, key); // 依赖收集
	        return Reflect.get(target, key);
	      },
	      set(target, key, value) {
	        const oldValue = target[key];
	        const result = Reflect.set(target, key, value);
	        
	        // 特殊处理数组的 length 属性
	        if (key === 'length' && Array.isArray(target)) {
	          // 触发 length 相关更新
	        } else {
	          // 判断是新增还是修改
	          if (!target.hasOwnProperty(key)) {
	            trigger(target, 'add', key);
	          } else if (oldValue !== value) {
	            trigger(target, 'set', key);
	          }
	        }
	        return result;
	      },
	      deleteProperty(target, key) {
	        const hadKey = target.hasOwnProperty(key);
	        const result = Reflect.deleteProperty(target, key);
	        if (hadKey) {
	          trigger(target, 'delete', key);
	        }
	        return result;
	      }
	    });
	  }
	  ```
	- Proxy多达13种拦截方法；
	  get、set、has、deleteProperty、ownKeys、getOwnPropertyDescriptor、
	  defineProperty、preventExtensions、getPrototypeOf、isExtensible、
	  setPrototypeOf、apply、construct
	- Proxy返回一个新对象，我们只操作新对象；
	- Proxy持续优化；
	- 不支持IE；
	
- Object.defineProperty
	- 支持IE9；
	
	- 只能遍历对象属性直接修改；
	
	- 不再优化；
	
	- 不能监听数组
	
	  - 为什么不监听数组下标
	
	    - 性能问题
	
	      - 数组包含大量元素，为每个索引都设置getter/setter会消耗大量内存
	      - 每次数组操作都要遍历整个数组，时间复制度为O(n)
	
	    - 下标不可控
	
	      - 数组长度动态变化，无法预先定义所有索引的响应式
	      - 数组操作（push/pop/shift/unshift）会改变索引位置
	
	    - 监听盲区
	
	      ```js
	      const arr = [1, 2, 3];
	      // Vue2 无法检测到这种直接索引赋值
	      arr[0] = 10; 
	      // Vue2 无法检测到数组长度变化
	      arr.length = 0;
	      ```
	
	    - 无法监听新增、删除
	
	      ```js
	      // 无法检测新增元素
	      arr[3] = 4;
	      // 无法检测元素删除
	      delete arr[0];
	      ```
## 对比总结

| 特性                    | `Object.defineProperty` | `Proxy`            |
| :---------------------- | :---------------------- | :----------------- |
| 数组索引访问/设置       | ❌ 无法监听              | ✅ 完全支持         |
| 数组方法操作 (push/pop) | ⚠️ 需重写方法            | ✅ 原生支持         |
| 数组长度变化            | ❌ 无法监听              | ✅ 支持 length 变更 |
| 新增/删除元素           | ❌ 无法监听              | ✅ 完全支持         |
| 性能表现                | ⚠️ 数组较大时性能差      | ✅ 高效             |
| 嵌套对象处理            | ⚠️ 需要递归初始化        | ✅ 惰性代理         |
| 浏览器兼容性            | ✅ IE9+                  | ❌ 不兼容 IE        |

### 服务端渲染

- 定义：在服务端渲染整个html片段，直接返回给客户端；
- 优点：有利于SEO
- 缺点：只支持beforeCreate，created钩子函数；服务端渲染的应用只能在node server运行环境；更多的服务器负载；
- nuxtjs：vue-server-renderer
### vue常用修饰符
- stop
	阻止冒泡
- prevent
	阻止默认事件（如：a标签的跳转）
- capture
	事件默认是由里往外冒泡；capture作用是由外往里捕获
- self
	点击事件绑定的本身才会触发事件
- once
	事件只执行一次
- passive
	相当于给onscroll事件添加一个.lazy修饰符（@scroll.passive="onScroll"）
- enter
- trim
	去除首尾空格
- number
	将v-model值转成数字
		22aa => 22
		aa22 => aa22（无效）
- lazy
	改变输入框的值时value不会改变；当光标离开输入框时，v-model绑定的值value才会改变
- sync
	配合this.$emit('update:xxx', data)
- native
	加在自定义组件的事件上，保证事件能执行
	

### keep-alive

- #### 源码 src/core/components/keep-alive.js

  ```js
  export default {
    name: 'keep-alive',
    abstract: true,
  
    props: {
      include: [String, RegExp, Array],
      exclude: [String, RegExp, Array],
      max: [String, Number]
    },
  
    created () {
      this.cache = Object.create(null)
      this.keys = []
    },
  
    destroyed () {
      for (const key in this.cache) {
        pruneCacheEntry(this.cache, key, this.keys)
      }
    },
  
    mounted () {
      this.$watch('include', val => {
        pruneCache(this, name => matches(val, name))
      })
      this.$watch('exclude', val => {
        pruneCache(this, name => !matches(val, name))
      })
    },
  
    render() {
      /* 获取默认插槽中的第一个组件节点 */
      const slot = this.$slots.default
      const vnode = getFirstComponentChild(slot)
      /* 获取该组件节点的componentOptions */
      const componentOptions = vnode && vnode.componentOptions
  
      if (componentOptions) {
        /* 获取该组件节点的名称，优先获取组件的name字段，如果name不存在则获取组件的tag */
        const name = getComponentName(componentOptions)
  
        const { include, exclude } = this
        /* 如果name不在inlcude中或者存在于exlude中则表示不缓存，直接返回vnode */
        if (
          (include && (!name || !matches(include, name))) ||
          // excluded
          (exclude && name && matches(exclude, name))
        ) {
          return vnode
        }
  
        const { cache, keys } = this
        /* 获取组件的key值 */
        const key = vnode.key == null
          // same constructor may get registered as different local components
          // so cid alone is not enough (#3269)
          ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
          : vnode.key
       /*  拿到key值后去this.cache对象中去寻找是否有该值，如果有则表示该组件有缓存，即命中缓存 */
        if (cache[key]) {
          vnode.componentInstance = cache[key].componentInstance
          // make current key freshest
          remove(keys, key)
          keys.push(key)
        }
          /* 如果没有命中缓存，则将其设置进缓存 */
          else {
          cache[key] = vnode
          keys.push(key)
          // prune oldest entry
          /* 如果配置了max并且缓存的长度超过了this.max，则从缓存中删除第一个 */
          if (this.max && keys.length > parseInt(this.max)) {
            pruneCacheEntry(cache, keys[0], keys, this._vnode)
          }
        }
  
        vnode.data.keepAlive = true
      }
      return vnode || (slot && slot[0])
    }
  }
  ```

- #### 源码解析

  - this.cache

    是一个对象，用来存储需要缓存的组件，格式如下

    ```js
    this.cache = {
        'key1':'组件1',
        'key2':'组件2',
        // ...
    }
    ```

  - pruneCacheEntry

    在组件销毁的时候执行pruneCacheEntry函数

    ```js
    function pruneCacheEntry (cache: VNodeCache, key: string, keys: Array<string>, current?: VNode) {
      const cached = cache[key]
      /* 判断当前没有处于被渲染状态的组件，将其销毁*/
      if (cached && (!current || cached.tag !== current.tag)) {
        cached.componentInstance.$destroy()
      }
      cache[key] = null
      remove(keys, key)
    }
    ```

  - 监听include & exclude变化

    ```js
    mounted () {
        this.$watch('include', val => {
            pruneCache(this, name => matches(val, name))
        })
        this.$watch('exclude', val => {
            pruneCache(this, name => !matches(val, name))
        })
    }
    ```

    如果include或exclude发生变化，即表示定义（不）需要缓存的组件的规则发生了变化，则执行pruneCache函数

    ```js
    function pruneCache (keepAliveInstance, filter) {
      const { cache, keys, _vnode } = keepAliveInstance
      for (const key in cache) {
        const cachedNode = cache[key]
        if (cachedNode) {
          const name = getComponentName(cachedNode.componentOptions)
          if (name && !filter(name)) {
            pruneCacheEntry(cache, key, keys, _vnode)
          }
        }
      }
    }
    ```

    

