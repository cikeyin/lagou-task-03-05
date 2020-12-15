##### 1、Vue 3.0 性能提升主要是通过哪几方面体现的？
答：
（1） 响应式系统升级
Vue3.0使用Proxy对象重写响应式系统

 -  可以监听动态新增的属性
 -  可以监听删除的属性
 -  可以监听数组的索引和length属性

（2）编译优化
Vue3.0中标记和提升所有的静态根节点，diff的时候只需要对比动态节点内容

（3）源码体积优化
- Vue3.0中移除了一些不常用的API
- Vue3.0对Tree-shaking支持的更好


##### 2、Vue 3.0 所采用的 Composition Api 与 Vue 2.x使用的Options Api 有什么区别？
答：
（1）Options Api  
- 包含描述组件选项（data、methods等）的对象
- 在开发复杂组件时，同一个功能逻辑代码被拆分到不同选项中，不利于维护和复用

（2）Composition Api 
- 一组基于函数的Api
- 可以更灵活的组织组件的逻辑



##### 3、Proxy 相对于 Object.defineProperty 有哪些优点？
答：
-  可以监听动态新增的属性，对于多层嵌套的属性，不会在初始化的时候处理下一级属性，会在访问属性时处理下一级属性
-  可以监听删除的属性
-  可以监听数组的索引和length属性


#####  4、Vue 3.0 在编译方面有哪些优化？
答：Vue2.x中通过标记静态根节点优化了diff的过程。Vue3.0中标记和提升所有的静态根节点，diff的时候只需要对比动态节点内容；缓存了事件处理函数



#####  5、Vue.js 3.0 响应式系统的实现原理？
答：
Vue 3.0使用 Proxy 对象重写了响应式系统，响应式系统主要有以下几个函数来组合完成的：

1. reactive
    (1) 接收一个参数，判断这参数是否是对象。不是对象则直接返回这个参数，不做响应式处理
	(2) 创建拦截器对象 handler, 设置 get/set/deleteProperty
		get
			收集依赖（track）
			返回当前 key 的值。
				如果当前 key 的值是对象，则为当前 key 的对象创建拦截器 handler, 设置 get/set/deleteProperty
				如果当前的 key 的值不是对象，则返回当前 key 的值
		set
			设置的新值和老值不相等时，更新为新值，并触发更新（trigger）
		deleteProperty
			当前对象有这个 key 的时候，删除这个 key 并触发更新（trigger）
	(3)返回 Proxy 对象

2. effect
	接收一个函数作为参数。作用是：访问响应式对象属性时去收集依赖   

3. track
	(1) 接收两个参数：target 和 key
	(2) 如果没有 activeEffect，则说明没有创建 effect 依赖
	(3) 如果有 activeEffect，则去判断 WeakMap 集合中是否有 target 属性，
		WeakMap 集合中没有 target 属性，则 set(target, (depsMap = new Map()))
		WeakMap 集合中有 target 属性，则判断 target 属性的 map 值的 depsMap 中是否有 key 属性.
			depsMap 中没有 key 属性，则 set(key, (dep = new Set()))
			depsMap 中有 key 属性，则添加这个 activeEffect

5. trigger
判断 WeakMap 中是否有 target 属性
WeakMap 中没有 target 属性，则没有 target 相应的依赖
WeakMap 中有 target 属性，则判断 target 属性的 map 值中是否有 key 属性，有的话循环触发收集的 effect()

```javascript
const isObject = val => val !== null && typeof val === 'object'
const convert = target => isObject(target) ? reactive(target) : target
const hasOwnProperty = Object.prototype.hasOwnProperty
const hasOwn = (target, key) => hasOwnProperty.call(target, key)

export function reactive(target) {
  if (!isObject(target)) return target

  const handler = {
    get(target, key, receiver) {
      // 收集依赖
      track(target, key)
      const result = Reflect.get(target, key, receiver)
      return convert(result)
    },
    set(target, key, value, receiver) {
      const oldValue = Reflect.get(target, key, receiver)
      let result = true
      if (oldValue !== value) {
        result = Reflect.set(target, key, value, receiver)
        // 触发更新
        trigger(target, key)
      }
      return result
    },
    deleteProperty(target, key) {
      const hadKey = hasOwn(target, key)
      const result = Reflect.deleteProperty(target, key)
      if (hadKey && result) {
        // 触发更新
        console.log('delete', key)
        trigger(target, key)
      }
      return result
    }
  }

  return new Proxy(target, handler)
}

let activeEffect = null
export function effect(callback) {
  activeEffect = callback
  callback() // 访问响应式对象属性，去收集依赖
  activeEffect = null
}

let targetMap = new WeakMap()
export function track(target, key) {
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

export function trigger(target, key) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return
  const dep = depsMap.get(key)
  if (dep) {
    dep.forEach(effect => {
      effect()
    })
  }
}

export function ref(raw) {
  // 判断 raw 是否是 ref 创建的对象，如果是的话直接返回
  if (isObject(raw) && raw.__v_isRef) {
    return
  }
  let value = convert(raw)
  const r = {
    __v_isRef: true,
    get value() {
      track(r, 'value')
      return value
    },
    set value(newValue) {
      if (newValue !== value) {
        raw = newValue;
        value = convert(raw)
        trigger(r, 'value')
      }
    }
  }
  return r
}

export function toRefs(proxy) {
  const ret = proxy instanceof Array ? new Array(proxy.length) : {}
  for (const key in proxy) {
    ret[key] = toProxyRef(proxy, key)
  }
  return ret
}

function toProxyRef(proxy, key) {
  const r = {
    __v_isRef: true,
    get value() {
      return proxy[key]
    },
    set value(newValue) {
      proxy[key] = newValue
    }
  }
  return r
}

export function computed(getter) {
  const result = ref()
  effect(() => (result.value = getter()))
  return result
}
```
