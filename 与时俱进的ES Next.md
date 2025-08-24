# 与时俱进的 ES Next

## 1. 数组 API

### Array.prototype.includes(value: any): Boolean

```
[1, 2, 3].includes(3) // true
```

- 判断数组中是否包含某元素，返回布尔值
- **其他判断元素的 API：**

```
[1, 2, 3].findIndex(i => i == 2) // 1
[1, 2, 3].find(i => i == 2)      // 2
[1, 2, 3].indexOf(2)             // 1
```

### 相等算法比较：indexOf VS includes

| 方法     | 比较算法        | 特点                          |
| -------- | --------------- | ----------------------------- |
| indexOf  | `===`           | `[NaN].indexOf(NaN) // -1`    |
| includes | `SameValueZero` | `[NaN].includes(NaN) // true` |

- `SameValueZero` 与 `===` 的区别：

  ```
  const set = new Set().add(0).add(NaN)
  set.has(-0) // true
  set.has(NaN) // true
  ```

### 总结

- **SameValueZero 是比 `===` 更宽松的 “严格相等”**：仅在 `NaN` 场景下比 `===` 更友好（认为 `NaN === NaN`），其他场景与 `===` 完全一致。
- **无需手动实现**：JavaScript 内置方法（`includes`、`Set.has`、`Map.has` 等）已默认使用 SameValueZero，直接使用这些方法即可满足大部分需求。
- **关键记忆点**：看到 `Array.includes()` 或 `Set.has()` 时，要想到它们用的是 SameValueZero，能正确识别 `NaN`。

## 2. 对象

### Object Spread（对象展开语法）

```
const a = { name: 'js' }
const b = { age: 20 }
const c = { ...a, ...b }
console.log(c) // { name: 'js', age: 20 }
```

> ```
> {...obj}` 等价于 `Object.assign({}, obj)
> ```

### Object Spread VS Object.assign()

- Object.assign 会触发 setter，并会修改目标对象：

```
const a = { name: 'js' }
const b = { age: 20 }
const c = Object.assign({}, a, b)
console.log(c) // { name: 'js', age: 20 }
```

> 推荐始终以空对象 `{}` 作为第一个参数，避免修改原对象

------

## 3. 箭头函数

### 不适合使用的场景

1. **构造函数的原型方法**

```
// 错误
Person.prototype = () => {...}
```

1. **需要 arguments 的场景**

```
const fn = () => console.log(arguments) // 不可用
```

1. **对象方法中**

```
const person = { 
  name: 'zhangsan',
  getName: () => console.log(this.name)
}
person.getName() // undefined
```

1. **动态回调**

```
btn.addEventListener('click', () => console.log(this === window)) // true
```

> 箭头函数没有独立 `this`，也没有 `arguments`，需要谨慎使用

------

## 4. Proxy 代理

### 对类构造函数的代理

```
class Person { constructor(name){ this.name = name } }

let proxyPersonClass = new Proxy(Person, {
  apply(target, context, args){
    throw new Error(`Function ${target.name} cannot be invoked without 'new'`)
  }
})

proxyPersonClass('zhangsan')       // Error
new proxyPersonClass('zhangsan')   // 正确
```

- 强制非构造调用转为 `new`：

```
let proxyPersonClass = new Proxy(Person, {
  apply(target, context, args){
    return new (target.bind(context, args))()
  }
})
proxyPersonClass('zhangsan')       // 正确
```

### assert 断言实现

```
const assert = new Proxy({}, {
  set(target, warning, value){
    if (!value) throw new Error(`assert: '${warning}' is error`)
  }
})
const data = '1'
assert['this is a number'] = typeof data === 'number' 
// Error: assert: 'this is a number' is error
```

### 装饰器（Decorators）及 autobind

- 问题示例：

```
const person = new Person('zhangsan')
const fn = person.getName
fn() // TypeError: Cannot read property 'name' of undefined
```

- 使用 `@autobind` 自动绑定 `this`：

```
class Person {
  constructor(name){ this.name = name }
  @autobind
  getName(){ return this.name }
}
const fn = new Person('zhangsan').getName
fn() // zhangsan
```

- autobind 实现原理：

```
function autobind(target, key, { value: fn, configurable, enumerable }){
  return {
    configurable,
    enumerable,
    get(){
      const boundFn = fn.bind(this)
      Object.defineProperty(this, key, {
        configurable: true,
        writable: true,
        enumerable: false,
        value: boundFn
      })
      return boundFn
    },
    set: createDefaultSetter(key)
  }
}
function createDefaultSetter(key){
  return function set(newValue){
    Object.defineProperty(this, key, {
      configurable: true,
      writable: true,
      enumerable: true,
      value: newValue
    })
    return newValue
  }
}
```

# 5. ES Next 新特性实战总结

## 5.1 可选链（Optional Chaining）`?.`

**作用**

- 安全访问对象深层属性或方法，避免 `TypeError: Cannot read property 'xxx' of undefined`

**示例**

```
const obj = { a: { b: 2 } }
console.log(obj?.a?.b)      // 2
console.log(obj?.x?.b)      // undefined
console.log(obj.a?.b?.c)    // undefined
```

- 结合函数调用：

```
const obj = { fn: () => 'ok' }
console.log(obj.fn?.())      // 'ok'
console.log(obj.noFn?.())    // undefined
```

------

## 5.2 空值合并运算符（Nullish Coalescing）`??`

**作用**

- 区分 `null/undefined` 与其他假值（0、''、false）
- 与逻辑 OR `||` 不同，`||` 会把所有假值替代，而 `??` 只替代 `null` 或 `undefined`

**示例**

```
const foo = null ?? 'default'   // 'default'
const bar = 0 ?? 42             // 0
const baz = '' ?? 'hello'       // ''
```

------

## 5.3 动态 import（Dynamic Import）

**作用**

- 按需加载模块，实现代码拆分（code-splitting）
- 返回一个 Promise

**示例**

```
// 按需加载模块
const loadModule = async () => {
  const { default: lodash } = await import('lodash')
  console.log(lodash.random(1, 10))
}
loadModule()
```

- 可结合条件加载：

```
if (isDev) {
  import('./dev-tools.js').then(module => module.init())
}
```

------

## 5.4 BigInt

**作用**

- 表示任意长度整数，解决 `Number.MAX_SAFE_INTEGER` 的精度问题

**示例**

```
const a = 9007199254740991n
const b = a + 10n
console.log(b) // 9007199254741001n
```

- 注意：
  - BigInt 不能与普通 Number 混合运算（需显式转换）

------

## 5.5 Promise.any & Promise.allSettled

### Promise.any

- 返回第一个成功的 Promise，如果全部失败，抛出 AggregateError

```
Promise.any([
  Promise.reject('err1'),
  Promise.resolve('ok2'),
  Promise.resolve('ok3')
]).then(console.log) // 'ok2'
```

### Promise.allSettled

- 等待所有 Promise 完成（无论成功或失败），返回状态数组

```
Promise.allSettled([
  Promise.resolve(1),
  Promise.reject(2)
]).then(console.log)
/*
[
  { status: 'fulfilled', value: 1 },
  { status: 'rejected', reason: 2 }
]
*/
```

------

## 5.6 逻辑赋值运算符（Logical Assignment）

- `||=`, `&&=`, `??=`

```
let a = null
a ??= 10
console.log(a) // 10

let b = true
b &&= false
console.log(b) // false
```

- 简化常用逻辑操作和默认值赋值场景

------

## 5.7 String 和 Array 新方法

### String

```
'hello'.at(-1)    // 'o'
'  hi  '.trimStart() // 'hi  '
'  hi  '.trimEnd()   // '  hi'
```

### Array

```
[1, 2, 3].at(-1)       // 3
[1, 2, 3].flatMap(x => [x, x*2]) // [1,2,2,4,3,6]
```
