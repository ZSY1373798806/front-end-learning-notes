```javascript
// TODO 实现resolve与reject
// TODO 状态不可变
// TODO throw
// TODO 实现then
// TODO then链式调用
// TODO Promise.all
// 接收一个Promise数组，数组中如有非Promise项，则此项当做成功
// 如果所有Promise都成功，则返回成功结果数组
// 如果有一个Promise失败，则返回这个失败结果
// TODO Promise.race
// 接收一个Promise数组，数组中如有非Promise项，则此项当做成功
// 哪个Promise最快得到结果，就返回那个结果，无论成功失败
// TODO Promise.allSettled
// 接收一个Promise数组，数组中如有非Promise项，则此项当做成功
// 把每一个Promise的结果，集合成数组，返回
// TODO Promise.any
// 接收一个Promise数组，数组中如有非Promise项，则此项当做成功
// 如果有一个Promise成功，则返回这个成功结果
// 如果所有Promise都失败，则报错

class MyPromise {
  constructor(executor) {
    this.initValue()
    // 初始化this指向
    this.initBind()

    // try...catch处理throw抛出的异常
    try {
      executor(this.resolve, this.reject)
    } catch (e) {
      this.reject(e)
    }
  }

  initBind() {
    // 初始化this
    this.resolve = this.resolve.bind(this)
    this.reject = this.reject.bind(this)
  }

  initValue() {
    this.PromiseResult = null
    this.PromiseState = 'pending'
    // 保存成功的回调
    this.onFulfilledCallbacks = []
    // 保存失败的回调
    this.onRejectedCallbacks = []
  }

  resolve(val) {
    // 状态不可变
    if (this.PromiseState !== 'pending') {
      return
    }
    this.PromiseResult = val
    this.PromiseState = 'fulfilled'

    // 执行保存的成功回调
    while (this.onFulfilledCallbacks.length) {
      this.onFulfilledCallbacks.shift()(this.PromiseResult)
    }
  }

  reject(reason) {
    if (this.PromiseState !== 'pending') {
      return
    }
    this.PromiseResult = reason
    this.PromiseState = 'rejected'

    // 执行保存的失败回调
    while (this.onRejectedCallbacks.length) {
      this.onRejectedCallbacks.shift()(this.PromiseResult)
    }
  }

  // 接收两个回调 onFulfilled，onRejected
  then(onFulfilled, onRejected) {

    const thenPromise = new MyPromise((resolve, reject) => {
      process.nextTick(() => {
        const resolvePromise = cb => {
          try {
            const x = cb(this.PromiseResult)
            if (x === thenPromise) {
              throw new Error('不能返回本身')
            }
            if (x instanceof MyPromise) {
              x.then(resolve, reject)
            } else {
              resolve(x)
            }
          } catch (err) {
            reject(err)
          }
        }

        // 参数校验，确保一定是函数
        onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : val => val
        onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason }

        if (this.PromiseState === 'fulfilled') {
          resolvePromise(onFulfilled)
        } else if (this.PromiseState === 'rejected') {
          resolvePromise(onRejected)
        } else if (this.PromiseState === 'pending') {
          // 如果状态为pending，暂存两个回调
          this.onFulfilledCallbacks.push(resolvePromise.bind(this, onFulfilled))
          this.onRejectedCallbacks.push(resolvePromise.bind(this, onRejected))
        }
      })
    });
    return thenPromise
  }

  static all(promises) {
    const result = []
    let count = 0
    return new MyPromise((resolve, reject) => {
      const addData = (index, value) => {
        result[index] = value
        count++
        if (count === promises.length) {
          resolve(result)
        }
      }
      promises.forEach((promise, index) => {
        if (promise instanceof MyPromise) {
          promise.then(res => {
            addData(index, res)
          }, err => {
            reject(err)
          })
        } else {
          addData(index, promise)
        }
      })
    })
  }

  static race(promises) {
    return new MyPromise((resolve, reject) => {
      promises.forEach((promise, index) => {
        if (promise instanceof MyPromise) {
          promise.then(res => {
            resolve(res)
          }, err => {
            reject(err)
          })
        } else {
          resolve(promise)
        }
      })
    })
  }

  static allSettled(promises) {
    return new MyPromise((resolve, reject) => {
      const result = []
      let count = 0
      const addData = ({ status, val }, index) => {
        const data = { status }
        status === 'fulfilled' ? data.value = val : data.reason = val
        result[index] = data
        count++
        if (count === promises.length) {
          resolve(result)
        }
      }
      promises.forEach((promise, index) => {
        if (promise instanceof MyPromise) {
          promise.then(res => {
            addData({ status: 'fulfilled', val: res }, index)
          }, err => {
            addData({ status: 'rejected', val: err }, index)
          })
        } else {
          addData({ status: 'fulfilled', val: promise }, index)
        }
      })
    })
  }

  static any(promises) {
    return new MyPromise((resolve, reject) => {
      let count = 0
      promises.forEach((promise, index) => {
        if (promise instanceof MyPromise) {
          promise.then(res => {
            resolve(res)
          }, err => {
            count++
            if (count === promise.length) {
              reject(err)
            }
          })
        } else {
          resolve(promise)
        }
      })
    })
  }
}


const p1 = () => {
  return new MyPromise((resolve) => {
    setTimeout(() => { resolve('1') })
  })
}

const p2 = () => {
  return new MyPromise((resolve, reject) => {
    reject('2')
  })
}

const p3 = () => {
  return new MyPromise((resolve) => {
    resolve('3')
  })
}

MyPromise.any([p1(), p2(), p3()]).then(res => { console.log(res) }, err => { console.error(err) })

```

