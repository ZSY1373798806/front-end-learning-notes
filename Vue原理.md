# 框架

## 关键词

- ### 双向绑定

- ### 依赖收集

- ### 发布订阅模式

- ### MVVM/MVC

- ### 虚拟DOM

- ### 虚拟DOM diff

- ### 模板编译



## 响应式框架基本原理

> 直观上，数据发生变化时，不再需要开发者手动更新视图，而视图会根据变化的数据自动进行更新。

### 步骤

1. #### 依赖收集

2. #### 数据劫持与代理

3. #### 发布订阅模式

### 实现

### 数据劫持与代理

```javascript
const data = {
  stage: 'gitChat',
  course: {
    title: '前端开发进阶',
    author: '张三',
    publishTime: '2020-08-05',
    keywords: ['js', '数据结构']
  }
}
```

#### Object.defineProperty()

```javascript
const observe = data => {
  if (!data || typeof data !== 'object') {
    return
  }
  Object.keys(data).forEach(key => {
    let currentValue = data[key]
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: false,
      get() {
        console.log(`getting ${key} value now, getting value is: `,currentValue)
        return currentValue
      },
      set(newValue) {
        currentValue = newValue
        console.log(`setting ${key} value now, setting value is: `,currentValue)
      }
    })
    // 递归遍历，实现深层数据拦截
    observe(currentValue)
  })
}
observe(data)
```

```javascript
data.stage
// getting stage value now, getting value is:  gitChat
```

```javascript
data.stage = 'github'
// setting stage value now, setting value is:  github
```

```javascript
data.course.title = '数据劫持'
// getting course value now, getting value is:  {...}
// setting title value now, setting value is:  数据劫持
```

```javascript
data.course.keywords.push('算法')
// getting course value now, getting value is:  {...}
// getting keywords value now, getting value is:  [ [Getter/Setter], [Getter/Setter] ]
```

> 我们监听到了data.stage和data.course.title的读写，以及data.course.keywords数组的读取；
>
> 而数组的push行为并没有被拦截；
>
> 这是因为Array.prototype上挂载的方法不能触发data.course.keywords属性值的setter，这不属于赋值操作，而是push API的调用操作。

###### Vue也存在同样的问题

> 解决方法：将数组的常用方法进行重写，覆盖掉原生的数组方法；
>
> 重写之后的数组方法需要能够被拦截。

实现：

```javascript
const arrExtend = Object.create(Array.prototype)
const arrMethods = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']
arrMethods.forEach(method => {
  const oldMethod = Array.prototype[method]
  const newMethod = function(...args) {
    oldMethod.apply(this, args)
    console.log(`${method} 方法被执行了`)
  }
  arrExtend[method] = newMethod
})
Array.prototype = Object.assign(Array.prototype, arrExtend)
```

```javascript
let array = [1,2,3]
array.push(4)
// push 方法被执行了
```

> 对数组的7个方法进行重写；
>
> 核心操作还是调用原生方法：oldMethod.apply(this, args)；
>
> 也可以在调用oldMethod.apply(this, args)前后插入任何我们需要的代码逻辑。

#### Proxy实现

```javascript
const observe = data => {
  if (!data || Object.prototype.toString.call(data) !== '[object Object]') {
    return
  }
  Object.keys(data).forEach(key => {
    let currentValue= data[key]
    if (typeof currentValue === 'object') {
      data[key] = new Proxy(currentValue, {
        set(target, property, value, receiver) {
          // 数组的push会引起length属性变化，push后会触发两次set操作；
          // 只需保留一次即可，prototype为length时忽略
          if (property !== 'length') {
            console.log(`setting ${key} value now, setting value is`, value)
          }
          return Reflect.set(target, property, value, receiver)
        }
      })
      observe(currentValue)
    } else {
      Object.defineProperty(data, key, {
        enumerable: true,
        configurable: false,
        get() {
          console.log(`getting ${key} value now, setting value is`, currentValue)
          return currentValue
        },
        set(newValue) {
          console.log(`setting ${key} value now, setting value is `, newValue)
          currentValue = newValue
        }
      })
    }
  })
}
observe(data)
```

```javascript
data.stage
// getting stage value now, setting value is gitChat
```

```javascript
data.stage = 'github'
// setting stage value now, setting value is  github
```

```javascript
data.course.title
// getting title value now, setting value is 前端开发进阶
```

```javascript
data.course.title = '数据劫持'
// setting course value now, setting value is 数据劫持
// setting title value now, setting value is  数据劫持
```

```javascript
data.course.keywords.push('算法')
// setting keywords value now, setting value is 算法
```

> 使用Proxy实现代理；
>
> 数据键值为基本类型时，依旧使用Object.defineProperty()；
>
> 对于键值为对象类型的，使用Proxy返回的新对象给data[key]重新赋值，这个新值得getter和setter将会被添加代理；并进行递归调用。

##### Object.defineProperty && Proxy

- ###### Object.defineProperty

  - 不能监听数组的变化，需要进行数组方法的重写；
  - 必须遍历对象的每个属性，且对于嵌套结构需要深层遍历；
  - 不再是优化重点。

- ###### Proxy

  - 代理是针对整个对象的，而不是对象的某个属性；
  - 只需要做一层代理就可以监听同级结构下的所有属性变化，对于深层结构，还是需要递归进行；
  - 支持代理数组的变化
  - 第二个参数handler对象除了get和set外，可以有13种拦截方法，更加强大；
  - 性能将会被底层持续优化。

### 模板编译实现

​		实现

​		使用正则+遍历，对#app节点下的内容进行替换，通过正则识别出模板变量，获取对应的数据。

- 案例1

```javascript
const data = {
  stage: 'gitChat',
  course: {
    title: '前端开发进阶',
    author: '张三',
    publishTime: '2020-08-05',
    keywords: ['js', '数据结构'],
  }
}
```

```html
<div id="app">
  <div>发布平台：{{stage}}</div>
  <div>发布课程：{{course.title }}</div>
  <div>发布人：{{ course.author}}</div>
  <div>发布时间：{{ course.publishTime   }}</div>
  <div>关键字：{{ course.keywords   }}</div>
</div>
```

![image-20210416230103348](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210416230128363.png)

```javascript
function compile(el, data) {
  // 创建文档碎片
  let fragment = document.createDocumentFragment()
  while (child = el.firstChild) {
    fragment.appendChild(child)
  }
  // 对el里面的内容进行替换
  function replace(fragment) {
    // 遍历节点
    Array.from(fragment.childNodes).forEach(node => {
      let textContent = node.textContent
      // 正则匹配 {{ xxxx }}
      let reg = /\{\{\s*(.*?)\s*\}\}/g
      if (node.nodeType === 3 && reg.test(textContent)) {
        const nodeTextContent = node.textContent
        const replaceText = () => {
          /**
                 * str.replace(reg, (param1, param2, param3, param4) => {...})
                 * reg：正则表达式 /\{\{\s*(.*?)\s*\}\}/g
                 * param1：匹配到的字符串 "{{stage}}"
                 * param2：对应替换的字符串 "stage"
                 * param3：替换的位置 5
                 * param4：整个要替换的字符串 "发布平台：{{stage}}"
                 */
          node.textContent = nodeTextContent.replace(reg, (matched, placeholder, param3, param4) => {
            const str = placeholder.split('.').reduce((prev, key) => {
              return prev[key]
            }, data)
            return str
          })
        }
        replaceText()
      }
      // 如果还有子节点，继续递归replace
      if (node.childNodes && node.childNodes.length) {
        replace(node)
      }
    })
  }
  replace(fragment)
  el.appendChild(fragment)
  return el
}
compile(document.querySelector('#app'), data)
```

![image-20210416230152400](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210416230152400.png)

> 使用fragment变量存储生成的真实HTML节点内容；
>
> 通过replace方法对{{ xxx }}进行数据替换，同时{{ xxx }}的表达式只会出现在nodeType为3的文本类型节点中；
>
> 因此对于node.nodeType===3&&reg.test(textContent)条件的情况，进行数据获取和填充；
>
> 借助字符串replace方法第二个参数进行一次性替换；
>
> 对于{{data.course.title}}等深层数据，通过reduce方法，获取正确的值；
>
> 对于存在子节点的节点，需要使用递归进行replace替换。

### 双向绑定实现

```html
<input v-model="stage"/>
```

![image-20210417221030137](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210417221030137.png)

```javascript
function compile(el, data) {
  // 创建文档碎片
  let fragment = document.createDocumentFragment()
  while (child = el.firstChild) {
    fragment.appendChild(child)
  }
  // 对el里面的内容进行替换
  function replace(fragment) {
    // 遍历节点
    Array.from(fragment.childNodes).forEach(node => {
      // ...
      // node.nodeType === 1 元素节点
      if (node.nodeType === 1) {
        let attributesArray = node.attributes
        Array.from(attributesArray).forEach(attr => {
          let attributeName = attr.name
          let attributeValue = attr.value
          console.log(Array.from(attributesArray), attributeName, attributeValue)
          if (attributeName.includes('v-')) {
            node.value = data[attributeValue]
            // console.log(eval(data))
          }
          node.addEventListener('input', e => {
            let newVal = e.target.value
            data[attributeValue] = newVal
            // ...
            // 更改数据源，触发setter
            // ...
          })
        })
      }
      // 如果还有子节点，继续递归replace
      if (node.childNodes && node.childNodes.length) {
        replace(node)
      }
    })
  }
  replace(fragment)
  el.appendChild(fragment)
  return el
}
compile(document.querySelector('#app'), data)
```

### 发布订阅模式

​		事件驱动理念即发布订阅模式（Pub/Sub模式）；

​		javascript本身就是事件驱动型语言，应用中的button进行事件绑定，点击后触发click事件。

> 优点：高内聚、低耦合。

- #### 简单案例

```javascript
class Notify {
  constructor() {
    // 订阅列表
    this.subscribers = []
  }
  // 订阅
  add(handler) {
    this.subscribers.push(handler)
  }
  //发布
  emit() {
    this.subscribers.forEach(subscriber => subscriber())
  }
}
const notify = new Notify()
notify.add(() => {
  console.log('emit here 1')
})
notify.add(() => {
  console.log('emit here 2')
})
notify.emit()
```

### MVVM

​		过程

> 首先对数据进行深度劫持和代理，对每个属性的getter和setter进行加工；
>
> 在模板初次编译时，解析指令（如v-model）；
>
> 并进行依赖收集{{ 变量 }}；
>
> 订阅数据的变化。

​		解释

1. 依赖收集过程：
   1. 当调用compiler的replace方法时，会读取数据进行模板变量的替换；
   2. 这时‘读取数据时’需要做一个标记，用来表示‘我依赖这一项数据’，因此需要订阅这个属性值得变化。

> Vue中定义一个Watcher类来表示观察订阅依赖；
>
> 加工指：在数据监听的getter中记录这个依赖，同时在setter触发数据变化时，执行依赖对应的相关操作；
>
> 最终触发模板中数据的变化。

​		流程图

![img](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210417230041838.png)

### 虚拟DOM

- 概念
  - 虚拟DOM就是用数据结构表示DOM结构，并没有真实的append到DOM上，因此称之为虚拟
- 收益
  - 操作数据结构（改变虚拟DOM对象）远比浏览器交互去操作DOM快很多。
- 注意
  - 虚拟DOM最终也要挂载到浏览器上成为真实的DOM节点，因此并不能使操作DOM的数量减少；
  - 但能够精确地获取最小的、最必要的操作DOM的集合。

##### 案例1

DOM结构

```html
<ul id="chapterList">
  <li class="chapter">chapter1</li>
  <li class="chapter">chapter2</li>
  <li class="chapter">chapter3</li>
</ul>
```

采用javascript对象结构表示

```javascript
const chapterListVirtualDom = {
  tagName: 'li',
  attributes: { id: 'chapterList' },
  children: [
    {
      tagName: 'li',
      attributes: { class: 'chapter' },
      children: ['chapter1']
    },
    {
      tagName: 'li',
      attributes: { class: 'chapter' },
      children: ['chapter2']
    },
    {
      tagName: 'li',
      attributes: { class: 'chapter' },
      children: ['chapter3']
    },
  ]
}
```

> tagName表示虚拟DOM对应真实DOM的标签类型；
>
> attributes是一个对象，表示DOM节点上的所有属性；
>
> children对应真实DOM的childNodes，其中childNodes的每一项又是类似的结构。

#### 虚拟DOM生成类的实现

​	用于生成虚拟DOM

```javascript
class Element {
  constructor(tagName, attributes = {}, children = []) {
    this.tagName = tagName
    this.attributes = attributes
    this.children = children
  }
}
function element(tagName, attributes, children) {
  return new Element(tagName, attributes, children)
}
```

```javascript
const chapterListVirtualDom = element('ul', { id: 'chapterList' }, [
  element('li', { class: 'chapter' }, ['chapter1']),
  element('li', { class: 'chapter' }, ['chapter2']),
  element('li', { class: 'chapter' }, ['chapter3']),
])
console.log(chapterListVirtualDom)
// {
//   tagName: 'li',
//   attributes: { id: 'chapterList' },
//   children: [
//     { tagName: 'li', attributes: { class: 'chapter' }, children: ['chapter1'] },
//     { tagName: 'li', attributes: { class: 'chapter' }, children: ['chapter2'] },
//     { tagName: 'li', attributes: { class: 'chapter' }, children: ['chapter3'] }
//   ]
// }
```

#### 实现setAttribute方法，对DOM节点进行属性设置

```javascript
const setAttribute = (node, key, value) => {
  switch (key) {
    case 'style':
      node.style.cssText = value
      break
    case 'value':
      let tagName = node.tagName || ''
      tagName = tagName.toLowerCase()
      if (tagName === 'input' || tagName === 'textarea') {
        node.value = value
      } else {
        node.setAttribute(key, value)
      }
      break
    default:
      node.setAttribute(key, value)
      break
  }
}
```

#### 实现Element类的render原型方法，根据虚拟DOM生成真实的DOM片段

```javascript
class Element {
  // ...
  render() {
    // 根据节点名称，创建对应的元素
    let element = document.createElement(this.tagName)
    let attributes = this.attributes
    // 遍历当前属性对象，并给对应的元素设置属性值
    for (let key in attributes) {
      setAttribute(element, key, attributes[key])
    }
    let children = this.children
    // 遍历子节点
    children.forEach(child => {
      // 判断当前节点是否为虚拟节点，若是则进行递归遍历；否则为为本节点，并直接创建文本节点
      let childElement = child instanceof Element 
      	? child.render() : document.createTextNode(child)
      element.appendChild(childElement)
    })
    return element
  }
}
```

#### *将真实DOM节点渲染到浏览器上*

```javascript
const renderDom = (element, target) => {
  target.appendChild(element)
}
```

#### 执行代码

```html
<div id="app"></div>
```

```javascript
const chapterListVirtualDom = element('ul', { id: 'chapterList' }, [
  element('li', { class: 'chapter' }, ['chapter1']),
  element('li', { class: 'chapter' }, ['chapter2']),
  element('li', { class: 'chapter' }, ['chapter3']),
])
const dom = chapterListVirtualDom.render()
const target = document.querySelector('#app')
renderDom(dom, target)
```

![image-20210419093535759](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210419093603399.png)

### 虚拟DOM diff算法

#### 实现

```javascript
const diff = (oldVirtualDom, newVirtualDom) => {
  let patches = []
  // 递归树，比较后的结果放到patches
  walkToDiff(oldVirtualDom, newVirtualDom, 0, patches)
  // 返回diff结果
  return patches
}
```

```javascript
let initialIndex = 0
/**
 * @param {旧VDOM} oldVirtualDom
 * @param {新VDOM} newVirtualDom
 * @param {nodeIndex} index
 * @param {闭包变量，记录diff结果} patches
 */
const walkToDiff = (oldVirtualDom, newVirtualDom, index, patches) => {
  let diffResult = []
  // 如果newVirtualDom不存在，说明该节点被移除，
  // 我们将type为REMOVE的对象推进diiResult变量，并记录index
  if(!newVirtualDom) {
    diffResult.push({
      type: 'REMOVE',
      index
    })
  }
  // 新旧节点都是文本节点，为字符串
  else if (typeof oldVirtualDom === 'string' && typeof newVirtualDom === 'string') {
    // 比较文本是否相同，如果不同则记录新的结果
    if (oldVirtualDom !== newVirtualDom) {
      diffResult.push({
        type: 'MODIFY_TEXT',
        data: newVirtualDom,
        index
      })
    }
  }
  // 新旧节点类型相同
  else if (oldVirtualDom.tagName === newVirtualDom.tagName) {
    // 比较属性是否相同
    let diffAttributeResult = {}
    // 旧节点属性与新节点不同
    for (let key in oldVirtualDom) {
      if (oldVirtualDom[key] !== newVirtualDom[key]) {
        diffAttributeResult[key] = newVirtualDom[key]
      }
    }
    // 旧节点中不存在新节点的属性
    for (let key in newVirtualDom) {
      // 旧节点不存在的新属性
      if (!oldVirtualDom.hasOwnProperty(key)) {
        diffAttributeResult[key] = newVirtualDom[key]
      }
    }
    if (Object.keys(diffAttributeResult).length > 0) {
      diffResult.push({
        type: 'MODIFY_ATTRIBUTES',
        diffAttributeResult
      })
    }
    // 若果存在子节点，遍历子节点
    oldVirtualDom.children.forEach((child, index) => {
      walkToDiff(child, newVirtualDom.children[index], ++initialIndex, patches)
    })
  }
  // 节点类型不同，旧节点直接被替换；直接将新的结果push
  else {
    diffResult.push({
      type: 'REPLACE',
      newVirtualDom
    })
  }
  // 旧节点不存在，直接将新的结果push
  if (!oldVirtualDom) {
    diffResult.push({
      type: 'REPLACE',
      newVirtualDom
    })
  }
  // diff结果
  if(diffResult.length) {
    patches[index] = diffResult
  }
}
```

#### 执行

```javascript
const chapterListVirtualDom1 = element('ul', { id: 'list' }, [
  element('li', { class: 'chapter' }, ['chapter1']),
  element('li', { class: 'chapter' }, ['chapter2']),
  element('li', { class: 'chapter' }, ['chapter3'])
])
const chapterListVirtualDom2 = element('ul', { id: 'list2' }, [
  element('li', { class: 'chapter2' }, ['chapter4']),
  element('li', { class: 'chapter2' }, ['chapter5']),
  element('lo', { class: 'chapter2' }, ['chapter6'])
])
const diffDom = diff(chapterListVirtualDom1, chapterListVirtualDom2)
console.log(diffDom)
```

![image-20210419150009512](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210419151116065.png)

### 小结

- 通过Element 类生成虚拟DOM；
- 通过diff方法对任意两个虚拟DOM进行对比，得到差异的数据数组

### 更新DOM节点 patch方法

```javascript
/**
 * @param {真实的DOM节点} node
 * @param {差异数据数组} patches
 */
const patch = (node, patches) => {
  let walker = {index: 0}
  walk(node, walker, patches)
}
const walk = (node, walker, patches) => {
  let currentPatch = patches[walker.index]
  let childNodes = node.childNodes
  // 递归调用
  childNodes.forEach(child => {
    walker.index ++
    walk(child, walker, patches)
  })
  // 调用doPatch方法进行更新
  if (currentPatch) {
    doPatch(node, currentPatch)
  }
}
const doPatch = (node, patches) => {
  patches.forEach(patch => {
    switch (patch.type) {
      // 修改节点属性
      case 'MODIFY_ATTRIBUTES':
        const attributes = patch.diffAttributeResult.attributes
        for (let key in attributes) {
          // 不为元素节点
          if (node.nodeType !== 1) {
            return
          }
          const value = attributes[key]
          if (value) {
            setAttribute(node, key, value)
          } else {
            node.removeAttribute(key)
          }
        }
        break
      // 修改节点文本
      case 'MODIFY_TEXT':
        node.textContent = patch.data
        break
      // 替换为新节点
      case 'REPLACE':
        let newNode = (patch.newVirtualDom instanceof Element) 
        	? patch.newVirtualDom.render() : document.createTextNode(patch.newVirtualDom)
        node.parentNode.replaceChild(newNode, node)
        break
      // 删除旧节点
      case 'REMOVE':
        node.parentNode.removeChild(node)
        break
      default:
        break
    }
  })
}
```

### 执行结果

```html
<div id="app"></div>
```

```javascript
const chapterListVirtualDom = element('ul', { id: 'chapterList' }, [
  element('li', { class: 'chapter' }, ['chapter1']),
  element('li', { class: 'chapter' }, ['chapter2']),
  element('li', { class: 'chapter' }, ['chapter3']),
])

const dom = chapterListVirtualDom.render()
const target = document.querySelector('#app')
renderDom(dom, target)
```

![image-20210419165644206](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210419165644206.png)

```javascript
const chapterListVirtualDom2 = element('ul', { id: 'chapterList1' }, [
  element('li', { class: 'chapter' }, ['chapter1']),
  element('li', { class: 'chapter' }, ['chapter5']),
  element('h3', { class: 'chapter2', style: 'color: #f00' }, ['chapter6']),
])
const diffDom = diff(chapterListVirtualDom, chapterListVirtualDom2)
patch(dom, diffDom)
```

![image-20210419165716740](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210419165716740.png)