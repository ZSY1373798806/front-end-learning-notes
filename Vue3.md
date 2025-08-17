### 生命周期

- #### 生命周期阶段

- 初始化

  - setup()

    代替beforeCreated和created，初始化响应式数据、方法等

  - 选项是API兼容

- 挂载

  - onBeforeMount
  - onMounted

- 更新

  - onBeforeUpdate
  - onUpdate

- 卸载

  - onBeforeUnmount
  - onUnMounted

- 其它

  - onErrorCaptured

    捕获子孙组件错误，可返回false阻止冒泡

  - onActivated/onDeactivated

    <keep-alive> 缓存组件激活/停用时调用

  - onServerPrefetch

    服务端渲染（SSR）期间异步获取数据

  - onRenderTracked

    跟踪响应式依赖

  - onRenderTriggered

    响应式变更触发

- #### 执行顺序

  - **挂载阶段**：`父onBeforeMount->子onBeforeMount->子onMounted->父onMounted`

  - **更新阶段**：`父onBeforeUpdate->子onBeforeUpdate->子onUpdated->父onUpdated`

  - **卸载阶段**：`父onBeforeUnmount->子onBeforeUnmount->子onUnmounted->父onUnmounted`

### 组件通信

- #### Props/Emits

  ```typescript
  interface Props {
     	prop1: string;
      prop2?: number;
  }
  const props = defineProps<Props>();
  
  interface CustomEmit {
      (e: 'click'): void;
    	(e: 'change', index: number)?: void;
  }
  const emit = defineEmits<CustomEmit>();
  ```

### Ref

#### 使用场景

- 管理基本类型数据
- 需要明确的数据引用（如传递到函数中仍然保持响应性）

#### 为什么使用.value

- 设计目的

  - 统一处理基本类型和对象

  - 基本类型无法通过Proxy代理，ref通过封装对象（{value: xxx}）实现响应式

- 底层实现

  ```js
  function ref(value) {
    return { 
      __v_isRef: true,
      get value() { track(this, 'value'); return value; },
      set value(newVal) { value = newVal; trigger(this, 'value'); }
    };
  }
  ```

#### 如何实现防抖

```js
function debouncedRef(value, delay = 200) {
    let timeout;
    return customRef((track, trigger) => ({
        get(){
            track();
            return value;
        }
        set(newVal){
            clearTimeout(timeout);
            timeout = setTimeout(() => {
                value = newVal;
                tigger();
            }, delay);
        }
    }))
}
// 使用
const text = debouncedRef('', 500);
```



### Reactive

#### 使用场景

- 管理复杂对象/数组，避免频繁使用.value
- 需要深层嵌套的响应式数据

#### reactive局限性

- 无法直接替换整个对象

  ```js
  let obj = reactive({ a: 1 });
  obj = { a: 2 }; // 响应式丢失！
  ```

  - 解决方案

    使用Object.assign或ref包裹

    ```js
    Object.assign(obj, { a: 2 }); // 保持响应性
    const objRef = ref({ a: 1 }); // 替换整个对象时响应式有效
    ```

- 如何解构reactive对象且保持响应性

  使用toRefs

  ```js
  const state = reactive({ count: 0, name: 'Vue' });
  const { count, name } = toRefs(state); // 保持响应性
  count.value++; // 生效
  ```

### 响应式原理

#### ref

- 基本类型

  通过 `RefImpl` 类实现，而 `RefImpl` 类内部使用了 **JavaScript 原生的 getter/setter 语法**（不是直接调用`Object.defineProperty`）。

- 对象类型

  内部转化为reactive代理

#### reactive

- 基于**Proxy**代理整个对象，递归处理嵌套数据
- **依赖收集**：在get时调用track收集依赖
- **触发更新**：在set时调用trigger通知更新

### ref & toRef & toRefs区别

- ref(value: T): Ref<T>

  创建一个响应式数据引用，接收一个初始值作为参数，并返回一个包含该值的响应式引用；ref是一个包装对象，它的.value属性用于访问和修改引用的值

  ```js
  import { ref } from 'vue';
  const count = ref(0); // 创建一个初始值为 0 的响应式引用
  console.log(count.value); // 输出: 0
  count.value++; // 修改引用的值
  console.log(count.value); // 输出: 1
  ```

- toRef(object: object, key: string | symbol): ToRef

  创建一个指向另一个响应式对象的响应式引用。接收一个响应式对象和其属性名作为参数，并返回一个指向该属性的响应式引用。`ToRef` 是一个只读的响应式引用

  ```js
  import { ref, reactive, toRef } from 'vue';
  const state = reactive({
    name: 'John',
    age: 30
  });
  const nameRef = toRef(state, 'name'); // 创建指向 state.name 的引用
  console.log(nameRef.value); // 输出: "John"
  state.name = 'Mike'; // 修改原始对象的属性值
  console.log(nameRef.value); // 输出: "Mike"
  nameRef.value = 'Amy'; // 修改引用的值
  console.log(state.name); // 输出: "Amy"
  ```

- toRefs(object: T): ToRefs<T>

  将一个响应式对象的所有属性转换为响应式引用。接收一个响应式对象作为参数，并返回一个包含所有属性的响应式引用对象。`ToRefs` 是一个对象，每个属性都是一个只读的响应式引用

  ```js
  import { reactive, toRefs } from 'vue';
  const state = reactive({
    name: 'John',
    age: 30
  });
  const refs = toRefs(state); // 将 state 中的所有属性转换为响应式引用
  console.log(refs.name.value); // 输出: "John"
  console.log(refs.age.value); // 输出: 30
  state.name = 'Mike'; // 修改原始对象的属性值
  console.log(refs.name.value); // 输出: "Mike"
  refs.age.value = 25; // 修改引用的值
  console.log(state.age); // 输出: 25
  ```

### 插槽slot

#### 作用域插槽

- 数据反向传递：允许子组件向父组件传递数据，实现内容渲染逻辑的定制

### Vue-Router

#### 组合式API

- useRouter()
  - 获取路由实例，代替this.$router
- useRoute()
  - 获取当前路由对象，代替this.$route
