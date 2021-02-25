# 生命周期

### 对比 2.x 的替换

```javascript
beforeCreate -> 使用 setup() 
created -> 使用 setup() 
beforeMount -> onBeforeMount 
mounted -> onMounted 
beforeUpdate -> onBeforeUpdate 
updated -> onUpdated 
beforeDestroy-> onBeforeUnmount 
destroyed -> onUnmounted 
activated -> onActivated 
deactivated -> onDeactivated 
errorCaptured -> onErrorCaptured

// 新增两个用于调试的生命周期钩子
onRenderTracked、onRenderTriggered
```

### 1、注册钩子函数

- 通过 createHook 函数完成注册（柯里化）

  - injectHook(lifecycle, hook, target)

    对 hook 进行一层封装，然后把它添加到当前组件实例上

### 2、执行钩子函数

- 通过组件副作用渲染函数 setupRenderEffect 的执行逻辑来看，会先通过 invokeArrayFns(bm) 执行 beforemount 钩子函数，然后把 mounted 钩子函数通过 queuePostRenderEffect 函数推入 postFlushCbs 中，等整个应用 render 完毕之后，再同步执行 flushPostFlushCbs 函数来调用 mounted 钩子函数。

- 嵌套组件执行 beforeMount 的顺序是先父后子，然后 mounted 的顺序是先子后父。

- 在执行 beforeUpdate 钩子函数前，DOM 还未更新，如果要手动移除事件监听的话，可以在这里注册。

- updated 钩子函数执行时，DOM 已经更新了。不要在它里面更改数据，导致无限递归。

- 嵌套组件执行 beforeUnmount 的顺序是先父后子，然后 unmount 的顺序是先子后父。

- errorCaptured 钩子函数的本质是捕获子孙节点的错误，当它返回 true 时就会停止向上传播。

- 当执行 track 函数时，会先进行依赖收集，然后就执行 onTrack 函数，即 onRenderTracked 钩子函数。

- 当执行 trigger 函数时，会先创建运行的 effects 集合，然后遍历执行的过程中就执行 onTrigger 函数，即 onRenderTrigger 钩子函数。

  

### Tips：

- 如果不依赖 DOM 的话，在 created 和 在 mounted 里面执行异步请求都是可以的。
- 父组件的更新不一定会导致子组件的更新，因为 Vue 的更新粒度是组件级别的。
- 组件在销毁阶段有一些内部创建的对象并不能自动清理，比如在组件内部创建了一个定时器，那么我们就可以在 beforeUnmount 钩子函数里面手动清理。
- 可以在根组件注册一个 errorCapture，用来进行信息统计和错误上报。

