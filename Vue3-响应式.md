# Reactive API

reactive 里面通过执行 createReactiveObject 方法来把 target 变成响应式对象

## createReactiveObject

createReactiveObject(target, false, mutableHandlers, mutableCollectionHandlers) 

1. 判断 target 是否是对象或数组，如果不是则直接返回

2. 判断 target 是否已经是响应式对象，如果是则直接返回该响应式对象

3. 通过 canObserve 方法判断 target 是否在白名单里

4. 利用 Proxy 创建响应式

   - mutableHandlers

     - **get 函数**

       1. 对特殊 key 做代理，比如：key == __v_raw 则返回 target

       2. 判断 target 是否是数组并且 key 是否命中了 arrayInstrumentations，如果是则执行 arrayInstrumentations 代理的函数，包括：includes、indexOf 、lastIndexOf ，同时还对数组的每个元素进行了依赖收集

       3. 如果不是数组，则通过 Reflect.get 方法求值，然后**执行 track 方法进行依赖收集**

          - track(target, type, key)

          - 在全局创建的 targetMap （WeakMap），键名是 target ，键值是 depsMap(Map) ，这个 depsMap 的键名是 target 的 key ，键值是 dep(Set)，dep 里面存储的是依赖 key 的副作用函数 effect。把当前激活的副作用函数 activeEffect 添加到 dep 里面

       4. 根据求值的结果，判断值是否是对象，如果是则递归调用 reactive

     - **set 函数**

       1. 通过 Reflect.set 求值
       2. 通过 trigger 方法派发通知
          - trigger(target, type, key, newValue)
          - 通过 targetMap 拿到 target 对应的 depsMap，通过 key 拿到对应的 dep
          - 根据 dep 创建运行的 effects 集合
          - 遍历执行 effects 集合

5. 给 target 的 __v_reactive 属性打标识（对同一个数据多次执行 reactive，会返回相同的响应式对象）

6. 返回响应式对象



# 副作用函数

```javascript
function effect(fn, options = EMPTY_OBJ)
```

- effect 函数里面通过执行 createReactiveEffect(fn, options) 来创建一个新的 reactiveEffect 函数，并添加一些属性
- 当 trigger 派发通知执行的 effect 方法就是这个 reactiveEffect 函数
  - 把全局的 activeEffect 指向当前的 reactiveEffect 
  - 执行被包装的 fn 函数
- 通过全局的 effectStack 栈，把当前的 reactiveEffect 入栈，待执行完毕后把它出栈，再把全局的 activeEffect  指向 effectStack[length - 1]，这样可以解决嵌套 effect 导致全局的 activeEffect 指向不准确的问题
- 在 reactiveEffect 入栈前，会执行 cleanup(effect) 方法清空它对应的依赖



# readonly

readonly 和 reactive 的区别是执行 createReactiveObject 方法时参数 isReadonly 不同

- readonlyHandlers 和 mutableHandlers 的主要区别在：get、set、deleteProperty

  readonly 的 get 不再进行依赖收集了



# ref

通过 createRef 方法把基础类型数据变成响应式

- 判断传入值是否已经是 ref，如果是就直接返回自身
- 判断传入值是否是对象或数组，如果是就把它转换成一个 reactive 对象，否则返回传入值
- 定义一个对象，对其 value 属性进行劫持
  - get value 则执行 track 进行依赖收集，并返回值
  - set value 则设置新值，并执行 trigger 派发更新

