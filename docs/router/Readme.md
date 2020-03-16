## Vue-router源码分析

### 使用
我们知道使用vue的时候，常常会伴随两个插件的使用`vue-router` `vuex`，下面我们分析下`vue-router`的源码实现，这样当我们使用或者遇到一些问题时，可以比较正确的解决。

```
import Vue from "vue";
import VueRouter from "vue-router";
import Home from "../views/Home.vue";

Vue.use(VueRouter);

const routes = [
  {
    path: "/",
    name: "Home",
    component: Home,
  },
  {
    path: "/about",
    name: "About",
    // route level code-splitting
    // this generates a separate chunk (about.[hash].js) for this route
    // which is lazy-loaded when the route is visited.
    component: () =>
      import(/* webpackChunkName: "about" */ "../views/About.vue"),
  },
];

const router = new VueRouter({
  mode: "history",
  base: process.env.BASE_URL,
  routes,
});

export default router;
```
可以看出，`vue-router`就是vue的插件，写法就是插件的写法，使用`vue.use`注册，然后`new VueRouter`出实例对象暴露出去。

### 注册 vue.use
首先我们看一下vue-router的入口
> 有的人会说看node_modules的时候不知道哪个是入口，那是因为那里的文件都是编译之后代码，我们可以先`git clone https://github.com/vuejs/vue-router.git`读取源码。

#### src/index.js
```
/* @flow */

import { install } from './install'
import { START } from './util/route'
import { assert } from './util/warn'
import { inBrowser } from './util/dom'
import { cleanPath } from './util/path'
import { createMatcher } from './create-matcher'
import { normalizeLocation } from './util/location'
import { supportsPushState } from './util/push-state'

import { HashHistory } from './history/hash'
import { HTML5History } from './history/html5'
import { AbstractHistory } from './history/abstract'

import type { Matcher } from './create-matcher'

export default class VueRouter {
  ...
}

function registerHook (list: Array<any>, fn: Function): Function {
  list.push(fn)
  return () => {
    const i = list.indexOf(fn)
    if (i > -1) list.splice(i, 1)
  }
}

function createHref (base: string, fullPath: string, mode) {
  var path = mode === 'hash' ? '#' + fullPath : fullPath
  return base ? cleanPath(base + '/' + path) : path
}

VueRouter.install = install
VueRouter.version = '__VERSION__'

if (inBrowser && window.Vue) {
  window.Vue.use(VueRouter)
}
```
我们看`src/index.js`的时候，其他的可以先不看，主要主意`export default class VueRouter {}`主要暴露的就是一个`VueRouter`构造函数，然后`VueRouter.install = install`挂载install方法，因为我们`vue.use(VueRouter)`的时候首先就是这些`install`方法，所以我们看看一下`install`的定义

#### src/install.js
```
import View from './components/view'
import Link from './components/link'

export let _Vue

export function install (Vue) {
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```
上面代码可知，install方法主要做的是使用`Vue.mixin`方法，这里我们可以学习的第一个知识点就是`beforeCreate`钩子函数。
> 因为在`beforeCreate`的时候，没有数据，所以我们可以做一些数据挂载的事情，我们以后写插件，也可以这么去做。

首先我们判断`if (isDef(this.$options.router)) {`是否没有传入，也就是
```
new Vue({
  router,
  store,
  render: h => h(App),
}).$mount("#app");
```
这里面有没有`router`实例对象，如果存在
```
this._routerRoot = this
this._router = this.$options.router
this._router.init(this)
```
去执行一些挂载和初始化。
```
Object.defineProperty(Vue.prototype, '$router', {
  get () { return this._routerRoot._router }
})

Object.defineProperty(Vue.prototype, '$route', {
  get () { return this._routerRoot._route }
})

Vue.component('RouterView', View)
Vue.component('RouterLink', Link)
```
上面这个比较重要，这里首先往`Vue.prototype`中添加两个`$router` `$route`属性，这也就是为什么我们在Vue页面中可以使用`this.$router` `this.$route`方法的原因。下面还给`RouterView` `RouterLink`做组件绑定，也就是我们页面访问的`<router-view></router-view>` `<router-link></router-link>`的原因。

**总结注册**
这里我们就知道了`install.js`做的一些事情了 主要使用vue.mixins混入一些初始化的东西，比如`this.$router ` `<router-view></router-view>`等

### 实例化new VueRouter
我们在调用的时候 会执行 `const router = new VueRouter({`这部分，然后返回一个router对象，挂载到new Vue上。 那么我们看看new VueRouter在做啥，我们从源码上知道 VueRouter 为class，会有些属性和方法。

```
export default class VueRouter {
  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    // 路由模式 根据不同没事 使用不用的API
    let mode = options.mode || 'hash'
    // history会使用HTML5History 而HTML5History并不是每个浏览器都支持 所以优雅降级为hash
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }

    // 不是浏览器的时候 也就是服务端渲染 使用abstract
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }

  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }

  get currentRoute (): ?Route {
    return this.history && this.history.current
  }

  init (app: any /* Vue component instance */) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )
    // app为new Router返回的实例router，因为new Router可以在页面中创建多次（尽管出现次数少）我们要将每个实例保存
    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null
    })

    // 如果只有一个new Router实例 那么下面就不执行
    if (this.app) {
      return
    }

    // 如果有多个  那么 this.app为最后一个
    this.app = app

    const history = this.history

    //根据不用model执行不同内容。
    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }

  beforeEach (fn: Function): Function {
    return registerHook(this.beforeHooks, fn)
  }

  beforeResolve (fn: Function): Function {
    return registerHook(this.resolveHooks, fn)
  }

  afterEach (fn: Function): Function {
    return registerHook(this.afterHooks, fn)
  }

  onReady (cb: Function, errorCb?: Function) {
    this.history.onReady(cb, errorCb)
  }

  onError (errorCb: Function) {
    this.history.onError(errorCb)
  }

  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    // $flow-disable-line
    if (!onComplete && !onAbort && typeof Promise !== 'undefined') {
      return new Promise((resolve, reject) => {
        this.history.push(location, resolve, reject)
      })
    } else {
      this.history.push(location, onComplete, onAbort)
    }
  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    // $flow-disable-line
    if (!onComplete && !onAbort && typeof Promise !== 'undefined') {
      return new Promise((resolve, reject) => {
        this.history.replace(location, resolve, reject)
      })
    } else {
      this.history.replace(location, onComplete, onAbort)
    }
  }

  go (n: number) {
    this.history.go(n)
  }

  back () {
    this.go(-1)
  }

  forward () {
    this.go(1)
  }

  getMatchedComponents (to?: RawLocation | Route): Array<any> {
    const route: any = to
      ? to.matched
        ? to
        : this.resolve(to).route
      : this.currentRoute
    if (!route) {
      return []
    }
    return [].concat.apply([], route.matched.map(m => {
      return Object.keys(m.components).map(key => {
        return m.components[key]
      })
    }))
  }

  resolve (
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
    location: Location,
    route: Route,
    href: string,
    // for backwards compat
    normalizedTo: Location,
    resolved: Route
  } {
    current = current || this.history.current
    const location = normalizeLocation(
      to,
      current,
      append,
      this
    )
    const route = this.match(location, current)
    const fullPath = route.redirectedFrom || route.fullPath
    const base = this.history.base
    const href = createHref(base, fullPath, this.mode)
    return {
      location,
      route,
      href,
      // for backwards compat
      normalizedTo: location,
      resolved: route
    }
  }

  addRoutes (routes: Array<RouteConfig>) {
    this.matcher.addRoutes(routes)
    if (this.history.current !== START) {
      this.history.transitionTo(this.history.getCurrentLocation())
    }
  }
}
```
当我们执行`new VueRouter`的时候先执行了`constructor`，具体我们内容我已经标注了，然后在Vue.mixin的时候中了`this._router.init(this)` 也就是`VueRouter init`方法。


<!--
#### src/history/base.js
我们根据不用model模式执行不同的API
```
switch (mode) {
  case 'history':
    this.history = new HTML5History(this, options.base)
    break
  case 'hash':
    this.history = new HashHistory(this, options.base, this.fallback)
    break
  case 'abstract':
    this.history = new AbstractHistory(this, options.base)
    break
  default:
    if (process.env.NODE_ENV !== 'production') {
      assert(false, `invalid mode: ${mode}`)
    }
}

HTML5History：
export class HTML5History extends History {}

HashHistory:
export class HashHistory extends History {}

AbstractHistory
export class AbstractHistory extends History {}

三种不同的路由模式 都继承了History基类
``` -->