# watch 实现原理

```javascript
function watch(source, cb, options) { 
  if ((process.env.NODE_ENV !== 'production') && !isFunction(cb)) { 
    warn(`\`watch(fn, options?)\` signature has been moved to a separate API. ` + 
      `Use \`watchEffect(fn, options?)\` instead. \`watch\` now only ` + 
      `supports \`watch(source, cb, options?) signature.`) 
  } 
  return doWatch(source, cb, options) 
} 
function doWatch(source, cb, { immediate, deep, flush, onTrack, onTrigger } = EMPTY_OBJ) { 
  // 标准化 source 
  // 构造 applyCb 回调函数 
  // 创建 scheduler 时序执行函数 
  // 创建 effect 副作用函数 
  // 返回侦听器销毁函数 
}   
```



### 1、标准化 source

根据 source 类型把它变成 getter 函数。

如果 source 是 reactive 类型的对象，则会自动递归遍历子属性从而触发依赖收集。（优化点：wach 的对象最好直接是一个 getter 函数）

### 2、构建回调函数

对 cb 做了一层封装

- 如果组件已经销毁，则直接返回不再执行
- 通过 runner 函数求最新值(实际是执行上一步的 getter 函数)
- 根据 deep 或者是新旧值发生了变化，执行 cb
- 把旧值更新为新值，方便下一次比对

### 3、构建 scheduler 函数

scheduler 的作用是根据某种调度方式来执行函数

watcher 是根据 options 配置里的 flush 属性来调度的

- flush == sync 时，同步执行回调函数
- flush == pre 时，如果组件已经挂载了则进入异步队列，在组件更新前执行；如果组件还没挂载则同步执行，在组件挂载前执行
- 没有配置 flush 时，进入异步队列，在组件更新后执行

### 4、创建 effect 副作用函数

上面提到的 runner 函数其实就是 watcher 内部创建的 effect 函数

- runner 是一个 computed effect 
- runner 的执行方式是 lazy 的
- runner 的返回结果，相当于执行了标准化后的 getter
- runner 创建后会在组件实例中记录这个 runner

### 5、返回销毁函数

- 执行 stop 方法让 runner 失活，并移除依赖
- 如果组件注册了 watcher，则移除组件 effects 对 runner 的依赖

# 异步任务队列

构建 scheduler 函数时，利用 flush 属性控制 watcher 的调度执行方式，当 flush 不是 sync 时，会把回调函数推入一个异步队列。

在一个 Tick 内，即使多次修改侦听的数据，它的回调函数也只会执行一次。如果每次更新数据都触发重新渲染，那么代价太高了，所以组件的更新过程用异步的方式来实现。

# watchEffect

### 与 watch 的区别

- 侦听的对象不同

  watchEffect 只侦听普通函数，而 watch 可以侦听一个或多个响应式对象以及 getter 函数

- 没有回调函数

- 立即执行

  watchEffect 在创建好后会立即执行函数，而 watch 需要配置 immediate = true









# 技巧

- 拷贝数组：var arr = [...new Set(target)]

- 检测循环更新：

  ```javascript
  const RECURSION_LIMIT = 100;
  function checkRecursiveUpdates(seen, fn) { // seen 是一个 Map 
      if (!seen.get(fn)) {
          seen.set(fn, 1)
      } else {
          let count = seen.get(fn);
          if (count > RECURSION_LIMIT) {
              throw new Error("超过最大循环次数")
          } else {
              seen.set(fn, count + 1);
          }
      }
  }
  ```

  