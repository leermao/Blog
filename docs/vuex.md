## vuex源码分析

### 使用
首先我们看看vuex的使用
```js
import Vue from "vue";
import Vuex from "vuex";

Vue.use(Vuex);

const store = new Vuex.Store({
  state: {},
  mutations: {},
  actions: {},
  modules: {},
});

new Vue({
  router,
  store,
  render: h => h(App),
}).$mount("#app");
```
在vue文件中 我们先引入`vuex`，然后注册`vue.use(vuex)`注册，然后使用`new Vuex.Store`实例化store，最后将store传到vue options中；

### index.js
首先我们查看vuex的源码发现
```
{
  "name": "vuex",
  "version": "3.1.3",
  "description": "state management for Vue.js",
  "main": "dist/vuex.common.js",
  "module": "dist/vuex.esm.js",
  ....
}
```
发现入口为打包文件，源入口为`src/index.js`
```
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```
vuex返回一个对象，其中为我们常用的一些方法。
当我们使用`vue.use(vuex)`安装注册vuex的时候，我们会调用install方法

##### src/store.js
```
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```
首先会判断Vue是否被重复调用，然后执行`applyMixin(Vue)`

##### src/mixin.js
```
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```
因为vue `1.x`和 `2.x`都是使用了同一个vuex，所以我们在初始化的时候判断一下版本，我们不看1.x，当vue版本为2.x的时候，我们是用Vue.mixin进行全局混入，然后在全局的`beforeCreate`生命周期的时候执行`vuexInit`函数。
在vuexInit函数中首先我们获取参数，判断参数中是否有`store`,如果有，挂载到this.$store上，这样我们每一个vue组件中都可以使用 this.$store获取store实例。
接下来我们看看store实例的内容，通过初始化我们发现store实例是通过new Vuex.Store创建的，所以我们看一下store
```
```