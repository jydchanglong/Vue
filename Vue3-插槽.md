插槽本质上是延时渲染，把父组件编写的插槽内容保存到一个对象上，然后把具体渲染 DOM 的代码用函数进行封装，当渲染子组件的时候，根据插槽名在对象中查找对应的函数，并执行这些函数从而进行真正的渲染。

## 1、**插槽初始化**

```vue
// template
<layout>
  <template v-slot:header>
    <h1>{{ header }}</h1>
  </template>
  <template v-slot:default>
    <p>{{ main }}</p>
    
  </template>
  <template v-slot:footer>
    <p>{{ footer }}</p>
  </template>
</layout>  
```

编译后的 render 函数

```javascript
import { toDisplayString as _toDisplayString, createVNode as _createVNode, resolveComponent as _resolveComponent, withCtx as _withCtx, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  const _component_layout = _resolveComponent("layout")

  return (_openBlock(), _createBlock(_component_layout, null, {
    header: _withCtx(() => [
      _createVNode("h1", null, _toDisplayString(_ctx.header), 1 /* TEXT */)
    ]),
    default: _withCtx(() => [
      _createVNode("p", null, _toDisplayString(_ctx.main), 1 /* TEXT */)
    ]),
    footer: _withCtx(() => [
      _createVNode("p", null, _toDisplayString(_ctx.footer), 1 /* TEXT */)
    ]),
    _: 1 /* STABLE */
  }))
}
```

这里 createBlock 函数通过 createVNode 创建 vnode，它的第三个参数就是一个**对象**。

createVNode 函数内部通过 normalizeChildren(vnode, children) 方法来处理子节点

- 标准化 Children
- 获取 vnode 节点类型 shapeFlag

确定了 shapeFlag 的类型，它会影响后续 patch 的执行逻辑，这里会走 processComponent 的逻辑。所以带有子节点插槽的组件和普通组件的渲染是一样的，都是通过递归的方式渲染子组件

渲染子组件的时候，会进入 setupComponent 的逻辑

```javascript
function setupComponent (instance, isSSR = false) {
  const { props, children, shapeFlag } = instance.vnode
  // 判断是否是一个有状态的组件
  const isStateful = shapeFlag & 4
  // 初始化 props
  initProps(instance, props, isStateful, isSSR)
  // 初始化插槽
  initSlots(instance, children)
  // 设置有状态的组件实例
  const setupResult = isStateful
    ? setupStatefulComponent(instance, isSSR)
    : undefined
  return setupResult
}
```

setupComponent 方法里面会执行 initSlots 函数来初始化插槽

```javascript
const initSlots = (instance, children) => {
  if (instance.vnode.shapeFlag & 32 /* SLOTS_CHILDREN */) {
    const type = children._
    if (type) {
      instance.slots = children // 后面就可以拿到插槽数据了
      def(children, '_', type)
    }
    else {
      normalizeObjectSlots(children, (instance.slots = {}))
    }
  }
  else {
    instance.slots = {}
    if (children) {
      normalizeVNodeSlots(instance, children)
    }
  }
  def(instance.slots, InternalObjectKey, 1)
}
```

## 2、插槽渲染

```vue
// template
<div class="layout">
  <header>
    <slot name="header"></slot>
  </header>
  <main>
    <slot></slot>
  </main>
  <footer>
    <slot name="footer"></slot>
  </footer>
</div>
```

编译后的 render 函数

```javascript
import { renderSlot as _renderSlot, createVNode as _createVNode, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(), _createBlock("div", { class: "layout" }, [
    _createVNode("header", null, [
      _renderSlot(_ctx.$slots, "header")
    ]),
    _createVNode("main", null, [
      _renderSlot(_ctx.$slots, "default")
    ]),
    _createVNode("footer", null, [
      _renderSlot(_ctx.$slots, "footer")
    ])
  ]))
}
```

子组件插槽部分的 DOM 主要通过 renderSlot 方法生成的

```javascript
function renderSlot(slots, name, props = {}, fallback) {
  let slot = slots[name]; // 这里 slots = instance.slots
  return (openBlock(),
    createBlock(Fragment, { key: props.key }, slot ? slot(props) : fallback ? fallback() : [], slots._ === 1 /* STABLE */
      ? 64 /* STABLE_FRAGMENT */
      : -2 /* BAIL */));
}

// slot 的值
_withCtx(() => [
  _createVNode("h1", null, _toDisplayString(_ctx.header), 1 /* TEXT */)
])
```

createBlock 方法创建了 vnode 节点，节点类型是 **Fragment**

```javascript
function withCtx(fn, ctx = currentRenderingInstance) {
  if (!ctx)
    return fn
  return function renderFnWithContext() {
    const owner = currentRenderingInstance
    setCurrentRenderingInstance(ctx)
    const res = fn.apply(null, arguments)
    setCurrentRenderingInstance(owner) // 这么做就是为了保证在子组件中渲染具体插槽内容时，它的渲染组件实例是父组件实例，这样也就保证它的数据作用域也是父组件的了。
    return res
  }
}
```

由于 vnode 的节点类型是 Fragment ，所以在 patch 挂载插槽时，会执行 processFragment 的逻辑

```javascript
const processFragment = (n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, optimized) => {
  const fragmentStartAnchor = (n2.el = n1 ? n1.el : hostCreateText(''))
  const fragmentEndAnchor = (n2.anchor = n1 ? n1.anchor : hostCreateText(''))
  let { patchFlag } = n2
  if (patchFlag > 0) {
    optimized = true
  }
  if (n1 == null) {
   //插入节点
// 先在前后插入两个空文本节点
    hostInsert(fragmentStartAnchor, container, anchor)
    hostInsert(fragmentEndAnchor, container, anchor)
    // 再挂载子节点
    mountChildren(n2.children, container, fragmentEndAnchor, parentComponent, parentSuspense, isSVG, optimized)
  } else {
    // 更新节点
  }
}
```

