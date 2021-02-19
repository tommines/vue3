### **1、Vue 3.0 性能提升主要是通过哪几方面体现的？**

**1.响应式系统升级**

​    Vue.js 2.x 中响应式系统的核心 defineProperty

​     Vue.js 3.0中使用Proxy 对象重写响应式系统

​              可以监听动态新增的属性

​              可以监听删除的属性

​              可以监听数组的索引和length属性

**2.编译优化**

Vue.js 2.x 中通过标记静态根节点，优化diff 的过程

Vue.js 3.0 中标记和提升所有的静态跟节点，diff 的时候只需要对比动态节点内容

   Fragments(升级 vetur 插件)

   静态提升

   Patch flag

   缓存事件处理函数



**3.源码体积的优化**

Vue.js 3.0 中移除了一些不常用的API

​     例如： inline-template,filter 等

 Tree-shaking



### 2、Vue 3.0 所采用的 Composition Api 与 Vue 2.x使用的Options Api 有什么区别？

**Options API**

   包含一个描述组件选项(data，methods, props等)的对象

   Options API 开发复杂组件，同一个功能逻辑的代码被拆分到不同选项

 **Composition API**

​    Vue.js 3.0 新增的一组 API

​     一组基于函数的API

​    可以更灵活的组织组件的逻辑



### 3、Proxy 相对于 Object.defineProperty 有哪些优点？

可以直接监听对象而非属性

可以直接监听数组的变化

有多达13种拦截方法，不限于apply,ownKeys,deleteProperty,has等等是Object.defineProperty 不具备的

返回的是一个新对象，我们可以只操作心得对象达到目的，而Object.defineProperty 只能遍历对象属性直接修改

作为新标准将受到浏览器厂商重点持续的性能优化，也就是传说中的新标准的性能红利

Object.defineProperty的优势如下:

兼容性好,支持IE9

### 4、Vue 3.0 在编译方面有哪些优化？

vue.js 3.x中标记和提升所有的静态节点，diff的时候只需要对比动态节点内容；

Fragments（升级vetur插件):

template中不需要唯一根节点，可以直接放文本或者同级标签

静态提升(hoistStatic),当使用 hoistStatic 时,所有静态的节点都被提升到 render 方法之外.只会在应用启动的时候被创建一次,之后使用只需要应用提取的静态节点，随着每次的渲染被不停的复用。

patch flag, 在动态标签末尾加上相应的标记,只能带 patchFlag 的节点才被认为是动态的元素,会被追踪属性的修改,能快速的找到动态节点,而不用逐个逐层遍历，提高了虚拟dom diff的性能。

缓存事件处理函数cacheHandler,避免每次触发都要重新生成全新的function去更新之前的函数tree shaking 通过摇树优化核心库体积,减少不必要的代码量

### 5、Vue.js 3.0 响应式系统的实现原理？

```javascript
const isObject = val => val !== null && typeof val === 'object'

const convert = target => isObject(target) ? reactive(target) : target

const hasOwnProperty = Object.prototype.hasOwnProperty

const hasOwn = (target, key) => hasOwnProperty.call(target, key)

/*
1.reactive
接受一个参数，是否是对象，不是返回
创建拦截器对象 handler, 设置 get/set/deleteProperty
get
收集依赖（track）
返回当前 key 的值。
如果当前 key 的值是对象，则为当前 key 的对象创建拦截器 handler, 设置 get/set/deleteProperty
如果当前的 key 的值不是对象，则返回当前 key 的值
set
设置的新值和老值不相等时，更新为新值，并触发更新（trigger）
deleteProperty
当前对象有这个 key 的时候，删除这个 key 并触发更新（trigger）
返回 Proxy 对象
2、effect: 接收一个函数作为参数。作用是：访问响应式对象属性时去收集依赖
3、track:
接收两个参数：target 和 key
如果没有 activeEffect，则说明没有创建 effect 依赖
如果有 activeEffect，则去判断 WeakMap 集合中是否有 target 属性，
WeakMap 集合中没有 target 属性，则 set(target, (depsMap = new Map()))
WeakMap 集合中有 target 属性，则判断 target 属性的 map 值的 depsMap 中是否有 key 属性
depsMap 中没有 key 属性，则 set(key, (dep = new Set()))
depsMap 中有 key 属性，则添加这个 activeEffect
4、trigger:
判断 WeakMap 中是否有 target 属性
WeakMap 中没有 target 属性，则没有 target 相应的依赖
WeakMap 中有 target 属性，则判断 target 属性的 map 值中是否有 key 属性，有的话循环触发收集的 effect()
5、 ref : 
判断 raw 是否是ref 创建的对象，如果是的话直接返回
r对象里 get方法 track方法判断后返回值，set方法判断新旧值，convert判断值是否不是对象是着返回，
最后调用trigger方法判断r对象是否在WeakMap里，如果存在，获取r对象里的值，循环访问响应式对象属性时去收集依赖
6、 toRefs :
判断值是否是数组,如果是就使用for in 循环导入到 toProxyRef，获取里面的值
7、toProxyRef：
r对象里get 返回传入的数组，与键，来得到对应的值 ，set 设置对应的键的新值
8、computed ：
将结果传到effect 收集依赖
*/

export function reactive (target) {

 if (!isObject(target)) return target

 const handler = {

  get (target, key, receiver) {

   // 收集依赖

   track(target, key)

   const result = Reflect.get(target, key, receiver)

   return convert(result)

  },

  set (target, key, value, receiver) {

   const oldValue = Reflect.get(target, key, receiver)

   let result = true

   if (oldValue !== value) {

    result = Reflect.set(target, key, value, receiver)

    // 触发更新

    trigger(target, key)

   }

   return result

  },

  deleteProperty (target, key) {

   const hadKey = hasOwn(target, key)

   const result = Reflect.deleteProperty(target, key)

   if (hadKey && result) {

   // 触发更新

   trigger(target, key)

   }

   return result

  }

 }

 return new Proxy(target, handler)

}

let activeEffect = null

export function effect (callback) {

 activeEffect = callback

 callback() // 访问响应式对象属性，去收集依赖

 activeEffect = null

}

let targetMap = new WeakMap()

export function track (target, key) {

 if (!activeEffect) return

 let depsMap = targetMap.get(target)

 if (!depsMap) {

  targetMap.set(target, (depsMap = new Map()))

 }

 let dep = depsMap.get(key)

 if (!dep) {

  depsMap.set(key, (dep = new Set()))

 }

 dep.add(activeEffect)

}


export function trigger (target, key) {

 const depsMap = targetMap.get(target)

 if (!depsMap) return

 const dep = depsMap.get(key)

 if (dep) {

  dep.forEach(effect => {

   effect()

  })

 }

}


export function ref (raw) {

 // 判断 raw 是否是ref 创建的对象，如果是的话直接返回

 if (isObject(raw) && raw.__v_isRef) {

  return

 }

 let value = convert(raw)

 const r = {

  __v_isRef: true,

  get value () {

   track(r, 'value')

   return value

  },

  set value (newValue) {

   if (newValue !== value) {

    raw = newValue

    value = convert(raw)

    trigger(r, 'value')

   }

  }

 }

 return r

}



export function toRefs (proxy) {

 const ret = proxy instanceof Array ? new Array(proxy.length) : {}

 for (const key in proxy) {

  ret[key] = toProxyRef(proxy, key)

 }

 return ret

}



function toProxyRef (proxy, key) {

 const r = {

  __v_isRef: true,

  get value () {

   return proxy[key]

  },

  set value (newValue) {

   proxy[key] = newValue

  }

 }

 return r

}



export function computed (getter) {

 const result = ref()

 effect(() => (result.value = getter()))

 return result

}
```

