## Node.js 模块循环引用（Circular Dependency）

**概念**
 当模块 A `require` 模块 B，同时模块 B 又 `require` 模块 A，就形成了循环引用。

**效果**

1. **导入的不完整对象**

   - Node.js 在加载模块时，会先将模块对象加入缓存，然后再执行模块代码。

   - 如果出现循环引用，模块加载到一半时被其他模块引用，则被引用的模块会得到一个 **部分初始化的对象**。

   - 例如：

     ```
     // a.js
     const b = require('./b');
     console.log('a -> b', b);
     module.exports = { name: 'moduleA' };
     
     // b.js
     const a = require('./a');
     console.log('b -> a', a);
     module.exports = { name: 'moduleB' };
     ```

     **输出可能为：**

     ```
     b -> a {}        // a.js 还没执行 module.exports 赋值
     a -> b { name: 'moduleB' }
     ```

2. **解决方式**

   - 尽量避免循环依赖
   - 将依赖放到函数内部延迟加载
   - 把公共逻辑抽离到第三个模块

------

## nextTick & setTimeout

**1. 执行时机对比**

| 特性     | process.nextTick               | setTimeout(fn, 0)                |
| -------- | ------------------------------ | -------------------------------- |
| 执行阶段 | 当前事件循环阶段结束后立即执行 | 下一轮事件循环的定时器阶段       |
| 优先级   | 高，优先于 Promise 微任务      | 较低，宏任务队列中等待执行       |
| 典型用途 | 在当前操作完成后立即执行回调   | 延迟执行、异步拆分任务、定时任务 |

**示例对比**

```
console.log('start');

process.nextTick(() => {
  console.log('nextTick');
});

setTimeout(() => {
  console.log('setTimeout');
}, 0);

console.log('end');
```

**输出顺序：**

```
start
end
nextTick
setTimeout
```

**2. 补充说明**

- **nextTick 会阻塞事件循环**：如果递归调用 nextTick，可能会导致 I/O 无法执行。
- **setImmediate vs setTimeout**：
  - `setImmediate` 在 I/O 回调之后执行
  - `setTimeout(fn, 0)` 在定时器阶段执行
  - 两者执行顺序不保证，但通常 I/O 回调后先执行 `setImmediate`