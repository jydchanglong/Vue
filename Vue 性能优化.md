# Vue 性能优化

### 子组件拆分

优化前

```vue
<template>
  <div :style="{ opacity: number / 300 }">
    <div>{{ heavy() }}</div>
  </div>
</template>

<script>
export default {
  props: ['number'],
  methods: {
    heavy () {
      const n = 100000
      let result = 0
      for (let i = 0; i < n; i++) {
        result += Math.sqrt(Math.cos(Math.sin(42)))
      }
      return result
    }
  }
}
</script>
```

优化后

```vue
<template>
  <div :style="{ opacity: number / 300 }">
    <ChildComp/>
  </div>
</template>

<script>
export default {
  components: {
    ChildComp: {
      methods: {
        heavy () {
          const n = 100000
          let result = 0
          for (let i = 0; i < n; i++) {
            result += Math.sqrt(Math.cos(Math.sin(42)))
          }
          return result
        },
      },
      render (h) {
        return h('div', this.heavy())
      }
    }
  },
  props: ['number']
}
</script>
```

优化思路：

1. 因为 Vue 更新是组件粒度的，通过把没有依赖的逻辑单独成组件，可以减少重复渲染
2. 可以利用 computed 属性缓存只需计算一次的结果

### 局部变量

优化前

```vue
<template>
  <div :style="{ opacity: start / 300 }">{{ result }}</div>
</template>

<script>
export default {
  props: ['start'],
  computed: {
    base () {
      return 42
    },
    result () {
      let result = this.start
      for (let i = 0; i < 1000; i++) {
        result += Math.sqrt(Math.cos(Math.sin(this.base))) + this.base * this.base + this.base + this.base * 2 + this.base * 3
      }
      return result
    },
  },
}
</script>
```

优化后：

```vue
<template>
  <div :style="{ opacity: start / 300 }">{{ result }}</div>
</template>

<script>
export default {
  props: ['start'],
  computed: {
    base () {
      return 42
    },
    result ({ base, start }) { // 优化点
      let result = start
      for (let i = 0; i < 1000; i++) {
        result += Math.sqrt(Math.cos(Math.sin(base))) + base * base + base + base * 2 + base * 3
      }
      return result
    },
  },
}
</script>
```

优化思路：在计算属性里可以通过缓存外部依赖数据，达到优化的目的。这是因为外部依赖的数据如果是响应式的，那么每次访问数据时都会触发它的 getter ，从而进行了依赖收集，多次访问后性能则会下降。

### 使用 `v-show` 复用 DOM

优化思路：v-show 仅仅是更改 DOM 的隐显值，所以开销比 v-if 要小很多。但 v-show 也要注意使用场景，它会把组件内的分支都渲染出来，对应的生命周期钩子函数都会执行。

### keep-alive

优化思路：keep-alive 包裹的组件会缓存 vnode 和 DOM，当切换路由视图时，可以直接从缓存里拿到对应的 vnode 和 DOM。但它会占用更多的内存来做缓存，是一种用空间换时间的优化。

### Deffered 组件延时渲染

优化前

```vue
<template>
  <div class="deferred-off">
    <VueIcon icon="fitness_center" class="gigantic"/>

    <h2>I'm an heavy page</h2>

    <Heavy v-for="n in 8" :key="n"/>

    <Heavy class="super-heavy" :n="9999999"/>
  </div>
</template>
```

优化后

```vue
<template>
  <div class="deferred-on">
    <VueIcon icon="fitness_center" class="gigantic"/>

    <h2>I'm an heavy page</h2>

    <template v-if="defer(2)">
      <Heavy v-for="n in 8" :key="n"/>
    </template>

    <Heavy v-if="defer(3)" class="super-heavy" :n="9999999"/>
  </div>
</template>

<script>
import Defer from '@/mixins/Defer'

export default {
  mixins: [
    Defer(),
  ],
}
</script>

// Defer 的 mixins
export default function(count = 10) {
  return {
	data() {
	  curPriority: 0
	},
	mounted() {
	  this.runPriority();
	},
	methods: {
	  runPriority() {
		const step = () => {
		  requestAnimationFrame(() => {
            this.curPriority++;
			  if(this.curPriority < count) {
				step();
		  	  }
          });
		}
		step();
	  },
	  defer(count) {
		return this.curPriority >= count
	  }
	}
  }
}

```

优化思路：Defer 主要是把一个组件的一次性渲染拆分成多次渲染，这种渐进式的渲染可以避免一次 render 过长造成的渲染卡顿现象。

### Non-reactive data

优化思路：把不是或者不用变成响应式的数据，单独拿出来，比如：放在 created() 钩子里面，这样可以在组件的上下文中达到共享数据的目的。