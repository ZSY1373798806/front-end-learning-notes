# 与时俱进的ES Next

## 数组API

### Array.prototype.includes(value: any): Boolean

```javascript
[1,2,3].includes(3) // true
```

> 判断数组中是否含有某一个元素，返回一个布尔值；

​		其他数组判断元素的api

 		[1, 2, 3].findIndex(*i* => *i* == 2) *// 2*

​		[1, 2, 3].find(*i* => *i* == 2) *// 2*

​		[1, 2, 3].indexOf(2) *// 1*



### 相等算法比较

#### 案例：indexOf VS includes

- ##### index

  > 采用的是===比较

  ```javascript
  [NaN].indexOf(NaN) // -1
  ```

- ##### includes

  > 采用SameValueZero()；
  >
  > 引擎内部的比较方式，没有对外接口；
  >
  > 实现方式采用了Map和Set。

  ```javascript
  [NaN].includes(NaN) // true
  ```

  ```javascript
  const set = new Set().add(0).add(NaN)
  set.has(-0) // false
  set.has(NaN) // true
  ```




## 对象

### Object Spread（对象展开语法）

```javascript
const a = { name: 'js' }
const b = {age: 20}
const c = {...a, ...b}
console.log(c)

// { name: 'js', age: 20 }
```

> 「object spread」：{... obj}等价于Object.assign（{}，obj）。

### Object Spread VS Object.assign()

- #### Object.assign()

  > Object.assign()修改了一个对象，因此它可以触发 ES6 setter；
  >
  > 使用时要保证始终将空对象{}作为第一个参数传递。

  ```javascript
  const a = { name: 'js' }
  const b = {age: 20}
  const c = Object.assign({}, a, b)
  console.log(c)
  
  // { name: 'js', age: 20 }
  ```

  

## 箭头函数

### 哪些场景下不适合使用ES6箭头函数

- #### 构造函数的原型方法上

  > 构造函数的原型方法需要通过this获得实例，箭头函数没有this。

  ```javascript
  // 错误语法
  Person.prototype = () => {...}
  ```

- #### 需要获取arguments时

  > 箭头函数不具有arguments，无法在函数体内访问这一特殊的伪数组。

- #### 使用对象方法时

  ```javascript
  const person = {
    name: 'zhangsan',
    getName: () => {
  		console.log(this.name)
    }
  }
  person.getName()
  
  // undefined
  ```

  > 上述例子中，getName函数内的this指向window，不符合其用意。

- #### 使用动态回调时

  ```javascript
  const btn = document.getElementById('btn')
  btn.addEventListener('click', () => {
    console.log(this === window)
  })
  
  // true
  ```

  > 当触发按钮事件时，会输出true，
  >
  > 事件绑定的函数this指向了window，无法获取该事件对象。



## Proxy代理

- ### 对象实例化（new关键字）

```javascript
class Person {
  constructor(name) {
    this.name = name
  }
}
```

```javascript
let proxyPersonClass = new Proxy(Person, {
  apply(target, context, args) {
    throw new Error(`error:Function ${target.name} cannot be invoked without 'new'`)
  }
})

proxyPersonClass('zhangsan') // Error: error:Function Person cannot be invoked without 'new'
new proxyPersonClass(' zhangsan') // 正确
```

> 对 Person 构造函数进行了代理，这样就可以防止非构造函数实例化的调用。

```javascript
let proxyPersonClass = new Proxy(Person, {
  apply(target, context, args) {
    console.log(target)
    return new (target.bind(context, args))()
  }
})

proxyPersonClass('zhangsan') // 正确
new proxyPersonClass(' zhangsan') // 正确
```

> 默认处理非构造函数实例化的调用，将其强制转换为new调用；
>
> 即使不使用new关键字，仍然可以得到new调用的实例。

- ### assert断言的实现

```javascript

const assert = new Proxy({}, {
  set(target, warning, value) {
    if (!value) {
      throw new Error(`assert: '${warning}' is error`)
    }
  }
})
const data = '1'
assert['this is a number'] = typeof data === 'number'

// Error: assert: 'this is a number' is error
```

- ### Decorators装饰器

  ​	装饰器就是给类添加或修改类的属性和方法

  以autobind类库的实现为例，介绍decorator的使用。

```javascript
class Person {
  constructor(name) {
    this.name = name
  }
  getName() {
    return this.name
  }
}
const person = new Person('zhangsan')
const fn = person.getName
fn()

// TypeError: Cannot read property 'name' of undefined
```

> 执行fn()时，this已经指向window，因此无法找到name属性。

使用autobind实现对this的绑定

```javascript
class Person {
  constructor(name) {
    this.name = name
  }
  @autobind
  getName() {
    return this.name
  }
}
const person = new Person('zhangsan')
const fn = person.getName
fn()

// zhangsan
```

autobind的实现

```javascript
// target：目标对象，这里是作用的Person中的函数和属性
// key：属性名称
// descriptor：属性原本的描述符
function autobind(target, key, { value: fn, configurable, enumerable }) {
  return {
    configurable,
    enumerable,
    get() {
      // 当使用get赋值时（const fn = person.getName）,
      // 赋值结果通过const boundFn = fn.bind(this)进行this绑定，并返回绑定this后的结果
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
function createDefaultSetter(key) {
  return function set(newValue) {
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

