## 指令的定义

本质上是一个 JavaScript 对象，在对象上有一些钩子函数，然后在合适的钩子函数里添加处理的逻辑

```javascript
const logDirective = {
  beforeMount() {
    console.log('log directive before mount')
  },
  mounted() {
     console.log('log directive mounted')
  },
  beforeUpdate() {
    console.log('log directive before update')
  },
  updated() {
    console.log('log directive updated')
  },
  beforeUnmount() {
    console.log('log directive beforeUnmount')
  },
  unmounted() {
    console.log('log directive unmounted')
  }
}
```



## 指令的注册

- 全局注册，是通过 app.directive 方法注册的，它是 app 对象上的方法

  ```javascript
  function createApp(rootComponent, rootProps = null) {
    const context = createAppContext()
    const app = {
      _component: rootComponent,
      _props: rootProps,
      directive(name, directive) { // 指令方法的实现
        if ((process.env.NODE_ENV !== 'production')) {
          validateDirectiveName(name)
        }
        if (!directive) {
          // 没有第二个参数，则获取对应的指令对象
          return context.directives[name]
        }
        if ((process.env.NODE_ENV !== 'production') && context.directives[name]) {
          // 重复注册的警告
          warn(`Directive "${name}" has already been registered in target app.`)
        }
        context.directives[name] = directive
        return app
      }
    }
    return app
  }
  ```

- 局部注册，是直接在组件对象中定义

  ```vue
  directives: {
    focus: {
      mounted(el) {
        el.focus()
      }
    }
  }
  ```

- 二者的区别就是全局注册的指令是保存在全局的context中，局部注册的指令保存在组件对象中

  

## 指令的使用

<input v-focus /> 编译后的 render 函数是

```javascript
import { resolveDirective as _resolveDirective, createVNode as _createVNode, withDirectives as _withDirectives, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  const _directive_focus = _resolveDirective("focus")
  return _withDirectives((_openBlock(), _createBlock("input", null, null, 512 /* NEED_PATCH */)), [
    [_directive_focus]
  ])
}
```

1、resolveDirective 方法内部通过调用 resolveAsset('directives', name) 函数获取对应的指令对象。resolveAsset 方法**先在**当前组件查找（局部注册），如果没有找到，则在全局 instance.appContext 查找（全局注册）

2、withDirectives 方法的实现

```javascript
function withDirectives(vnode, directives) {
  const internalInstance = currentRenderingInstance
  if (internalInstance === null) {
    (process.env.NODE_ENV !== 'production') && warn(`withDirectives can only be used inside render functions.`)
    return vnode
  }
  const instance = internalInstance.proxy
  const bindings = vnode.dirs || (vnode.dirs = [])
  for (let i = 0; i < directives.length; i++) {
    let [dir, value, arg, modifiers = EMPTY_OBJ] = directives[i]
    if (isFunction(dir)) {
      dir = {
        mounted: dir,
        updated: dir
      }
    }
    bindings.push({
      dir,
      instance,
      value,
      oldValue: void 0,
      arg,
      modifiers
    })
  }
  return vnode
}
```

3、在元素的生命周期中是如何运行指令里的钩子函数

- 元素的挂载是通过 mountElement 方法实现的

  ```javascript
  const mountElement = (vnode, container, anchor, parentComponent, parentSuspense, isSVG, optimized) => {
    let el
    const { type, props, shapeFlag, dirs } = vnode
    // 创建 DOM 元素节点
    el = vnode.el = hostCreateElement(vnode.type, isSVG, props && props.is)
    if (props) {
      // 处理 props，比如 class、style、event 等属性
    }
    if (shapeFlag & 8 /* TEXT_CHILDREN */) {
      // 处理子节点是纯文本的情况
      hostSetElementText(el, vnode.children)
    } else if (shapeFlag & 16 /* ARRAY_CHILDREN */) {
      // 处理子节点是数组的情况，挂载子节点
      mountChildren(vnode.children, el, null, parentComponent, parentSuspense, isSVG && type !== 'foreignObject', optimized || !!vnode.dynamicChildren)
    }
    if (dirs) {
      invokeDirectiveHook(vnode, null, parentComponent, 'beforeMount')
    }
    // 把创建的 DOM 元素节点挂载到 container 上
    hostInsert(el, container, anchor)
  
    if (dirs) {
      queuePostRenderEffect(()=>{ 
        invokeDirectiveHook(vnode, null, parentComponent, 'mounted')
      })
    }
  }
  ```

  它的内部在执行 hostInsert 前调用了元素指令的 beforeMount 钩子，在元素挂载后执行指令的 mounted 钩子

- 钩子函数的执行是通过 invokeDirectiveHook 方法实现的

  ```javascript
  function invokeDirectiveHook(vnode, prevVNode, instance, name) {
    const bindings = vnode.dirs
    const oldBindings = prevVNode && prevVNode.dirs
    for (let i = 0; i < bindings.length; i++) {
      const binding = bindings[i]
      if (oldBindings) {
        binding.oldValue = oldBindings[i].value
      }
      const hook = binding.dir[name]
      if (hook) {
        callWithAsyncErrorHandling(hook, instance, 8 /* DIRECTIVE_HOOK */, [
          vnode.el,
          binding,
          vnode,
          prevVNode
        ])
      }
    }
  }
  ```

  它的内部通过遍历 vnode.dirs 数组，拿到每个 hook ，再传递一些参数，最后再执行

- mounted 钩子函数的执行被包裹在 queuePostRenderEffect 方法里面，这样和组件的初始化过程执行 mounted 钩子函数一样（在整个应用 render 完毕后，同步执行 flushPostFlushCbs 的时候，执行元素指令的 mounted 钩子函数）

- 元素更新是通过 patchElement 实现的

  ```javascript
  const patchElement = (n1, n2, parentComponent, parentSuspense, isSVG, optimized) => {
    const el = (n2.el = n1.el)
    const oldProps = (n1 && n1.props) || EMPTY_OBJ
    const newProps = n2.props || EMPTY_OBJ
    const { dirs } = n2
    // 更新 props
    patchProps(el, n2, oldProps, newProps, parentComponent, parentSuspense, isSVG)
    const areChildrenSVG = isSVG && n2.type !== 'foreignObject'
  
    if (dirs) {
      invokeDirectiveHook(n2, n1, parentComponent, 'beforeUpdate')
    }
    // 更新子节点
    patchChildren(n1, n2, el, null, parentComponent, parentSuspense, areChildrenSVG)
  
    if (dirs) {
      queuePostRenderEffect(()=>{
        invokeDirectiveHook(vnode, null, parentComponent, 'updated')
      })
    }
  }
  ```

  在更新子节点前，执行了指令的 beforeUpdate 钩子，更新子节点后执行 updated 钩子

- 卸载元素是通过 unmount 方法实现的

  ```javascript
  const unmount = (vnode, parentComponent, parentSuspense, doRemove = false) => {
      ...
      if (shouldInvokeDirs) {
        invokeDirectiveHook(vnode, null, parentComponent, 'beforeUnmount')
      }
      // 卸载子节点
      // 移除 DOM 节点
      if ((vnodeHook = props && props.onVnodeUnmounted) || shouldInvokeDirs) {
          queuePostRenderEffect(() => {
            vnodeHook && invokeVNodeHook(vnodeHook, parentComponent, vnode)
            if (shouldInvokeDirs) {
              invokeDirectiveHook(vnode, null, parentComponent, 'unmounted')
            }
          }, parentSuspense)
      }
      ...
  }
  ```

  

















