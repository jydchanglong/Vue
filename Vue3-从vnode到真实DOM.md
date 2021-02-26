# 初始化

### 通过 createApp() 入口函数：

#### 1、创建 app 对象

- 通过 ensureRenderer().createApp() 方法来创建 app 对象

- ensureRenderer 方法内部创建了一个渲染器对象

  ```js
  return {
    render,
    createApp: createAppAPI(render)
  }
  ```

  createAppAPI 方法内部创建了 app 对象，它包括了 mount 方法。

  在 mount 方法里通过 createVNode 方法创建了根组件的 vnode，然后利用 render 渲染器渲染 vnode，最后返回的是 vnode.component.proxy。		

#### 2、重写 mount 方法

```javascript
app.mount = (containerOrSelector) => {
  const container = normalizeContainer(containerOrSelector)
  if (!container)
    return
  const component = app._component
  if (!isFunction(component) && !component.render && !component.template) {
    component.template = container.innerHTML
  }
  container.innerHTML = ''
  return mount(container)
}
```

重写 mount 方法是为了支持跨平台渲染，标准的跨平台渲染流程是：创建 vnode，渲染 vnode。

- 通过 normalizeContainer 方法标准化容器
- 判断组件对象有没有 render 或 template，如果没有则取容器里的 innerHTML 作为组件模板内容
- 清空容器的 innerHTML 
- 走标准的组件渲染流程

# 渲染流程

### 1、创建 vnode

#### vnode 的优势

- 抽象化
- 跨平台

#### createVNode 函数

```javascript
const vnode = createVNode(rootComponent, rootProps)
```

- 标准化 props 
- 对 vnode 类型信息编码（shapeFlag）
- 创建 vnode 对象
- 标准化子节点

### 2、渲染 vnode

```javascript
render(vnode, rootContainer)
```

- 判断是否有 vnode，如果没有则执行 unmount 方法销毁组件
- 如果有 vnode，则执行 patch 方法创建或更新组件
- 把 vnode 缓存到容器的 _vnode 属性

### 3、patch 函数

```javascript
const patch = (n1, n2, container, anchor = null, parentComponent = null, parentSuspense = null, isSVG = false, optimized = false)
```

参数：n1 代表旧的 vnode；n2 代表新的 vnode；container 表示容器。

- 首先判断新旧 vnode 的节点类型是否相同，如果不同则销毁旧节点
- 然后根据新 vnode 的节点类型走不同的处理逻辑

##### 对组件的处理 processComponent

- n1 == null，走挂载组件的逻辑：mountComponent
  - 创建组件实例：createComponentInstance
  - 设置组件实例：setupComponent
  - 设置并**运行**带副作用的渲染函数：setupRenderEffect
    - 它会创建响应式的副作用渲染函数，内部通过 effect 函数创建了一个副作用渲染函数 componentEffect
    - 它内部会判断是初始渲染，还是更新组件
      - 初始渲染
        - 通过 renderComponentRoot(instance) 方法生成子树 vnode --> subTree
        - 把 subTree 通过 patch 挂载到 container 中
        - 设置 instance.isMounted = true
      - 更新组件
        - 更新组件 vnode 节点信息 updateComponentPreRender
          - 新组件 vnode 的 component 属性指向组件实例
          - 组件实例的 vnode 属性指向新组件 vnode
          - 清空组件实例的 next 属性
          - 更新 props
          - 更新 slots
        - 渲染新的子树 vnode：renderComponentRoot(instance)
        - 根据新旧子树 vnode 做 patch 更新
- 否则走更新组件的逻辑：updateComponent
  - 通过 shouldUpdateComponent 函数，判断是否需要更新子组件
    - 把新的子组件 vnode 赋值给 instance.next
    - 把子组件从更新队列里移除 invalidateJob(instance.update)，这样可以避免子组件由于自身数据变化导致的重复更新
    - 执行子组件的副作用渲染函数 instance.update
  - 如果不需要更新，则只复制属性

##### 对元素的处理 processElement

- n1 == null，走挂载元素的逻辑 mountElement
  - 通过 hostCreateElement 方法创建 DOM 元素
    - 通过调用原生方法 document.createElement 创建 DOM
  - 添加 props 属性
  - 根据子节点类型进行对应的处理
    - 子节点类型是纯文本的情况 hostSetElementText()
      - el.textContent = text
    - 子节点类型是数组的情况 mountChildren
      - 遍历拿到每个 child，递归调用 patch 挂载 child
  - 通过 hostInsert 方法把创建的 DOM 元素挂载到 container 上
    - parent.insertBefore
    - parent.appendChild
- n1 != null，走更新元素的逻辑 patchElement
  - 通过 patchProps 更新 props
  - 通过 patchChildren 更新子节点
    - 如果新旧节点类型不同，先删除旧节点 unmountChildren
    - 如果新旧节点都是数组，则通过 patchKeyedChildren 函数做完整的 diff



## Tips

- Vue 是通过递归遍历 patch 方法完成构造 DOM 树的，是一种深度优先遍历的方式
- Vue 的挂载顺序是先子节点后父节点
- 组件更新有两种场景
  - 组件本身数据变化
    - 这个情况下 next = null
  - 父组件数据变化
    - 父组件在更新过程中遇到子组件节点，先判断子组件是否需要更新，如果是则主动调用子组件的重新渲染方法，这时 next = 新的子组件 vnode
- processComponent 处理组件 vnode 本质上是判断子组件是否需要更新，如果需要则递归执行子组件的副作用渲染函数来更新

