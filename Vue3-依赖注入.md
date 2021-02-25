# 依赖注入

### provide API

```javascript
function provide(key, value) {
  let provides = currentInstance.provides;
  const parentProvides = currentInstance.parent && currentInstance.parent.provides
  if (provides === parentProvides) {
    provides = currentInstance.provides = Object.create(parentProvides)
  }
  provides[key] = value;
}
```

在默认情况下，当前组件实例的 provides 对象继承自父组件实例的 provides 对象。当组件自己也要提供 provides 时，它会通过 Object.create(parentProvides) 来继承父级的原型对象，这样在子组件需要 inject 的地方可以通过原型链查找直接父组件的 provides。

组件实例可以覆盖和父组件相同 key 的数据。

### inject API

```javascript
function inject(key, defaultValue) {
  const instance = currentInstance || currentRenderingInstance;
  if (instance) {
  	const provides = instance.provides;
    if (key in provides) {
      return provides[key]    
    } else if (arguments.length > 1) {
      return defaultValue         
    } else {
        warn("没有这个 key")
    }
  }
}
```

# 依赖注入和模块化共享数据的差异

- 作用域不同

  - 依赖注入的作用域是局部范围的，它只能给后代组件利用

  - 模块化数据是全局范围的，任何地方都可以引用

- 数据来源不同

  - 依赖注入的数据只来源于父组件，所以可以直接使用
  - 要使用模块化数据，就必须清楚它是在哪定义的

- 上下文不同

  - 提供依赖注入的数据的组件上下文就是组件实例
  - 模块化数据是没有上下文的

# 依赖注入的缺陷

组件耦合严重，代码重构困难，在普通应用程序里面不推荐，但是在组件库里面可以使用。

# Tips

this.$parent 是一种强耦合的方式来获取父组件实例，非常不利于代码重构，慎用。