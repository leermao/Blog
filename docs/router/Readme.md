## Vue-router源码分析之matcher

### 使用
我们知道使用vue的时候，常常会伴随两个插件的使用`vue-router` `vuex`，下面我们分析下`vue-router`的源码实现，这样当我们使用或者遇到一些问题时，可以比较正确的解决。

```
import Vue from "vue";
import VueRouter from "vue-router";
import Home from "../views/Home.vue";

Vue.use(VueRouter);

const routes = [
  {
    path: "/home",
    name: "Home",
    component: Home,
    children: [
      {
        path: "/test",
        name: "Home1",
        component: Home,
      },
    ],
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
    // app为new Router返回的实例router，因为new Router可以在页面中创建多次（尽管出现次数少）我们要将每个实例保存
    this.apps.push(app)

    // 如果多次注册new Router实例 那么下面就不执行，第一次执行都注册
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

  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {

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

在执行`constructor`的时候，会对matcher做创建 `this.matcher = createMatcher(options.routes || [], this)`

```
src/create-matcher.js


export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)

  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }

  function match() {
    ...
  }

  return {
    match,
    addRoutes
  }
}
```
在执行`createMatcher`的时候 主要是调用`createRouteMap`
```
export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>
): {
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>
} {


  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })

  return {
    pathList,
    pathMap,
    nameMap
  }
}


function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  const { path, name } = route

  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props:
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props }
  }

  if (route.children) {
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }

  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }


  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      warn(
        false,
        `Duplicate named routes definition: ` +
          `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
```
我将伪代码放到了上面， `createRouteMap`主要返回`pathList pathMap nameMap`从明治我们知道，这三个变量主要是存储 **pach map**的，根据path返回内容 主要方法就是`routes.forEach`递归遍历执行`addRouteRecord`，目的是让children扁平化
```
routes就是我们传入的内容

const routes = [
  {
    path: "/home",
    name: "Home",
    component: Home,
    children: [
      {
        path: "/test",
        name: "Home1",
        component: Home,
      },
    ],
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
```
而`addRouteRecord`函数就是为往`pathList pathMap nameMap`存储 **RouteRecord**
```
const record: RouteRecord = {
  path: normalizedPath,
  regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
  components: route.components || { default: route.component },
  instances: {},
  name,
  parent,
  matchAs,
  redirect: route.redirect,
  beforeEnter: route.beforeEnter,
  meta: route.meta || {},
  props:
    route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props }
}
```
`RouteRecord`对象中包含了处理过的一些内容，方便我们使用。

这样处理`pathMap`返回的数据 伪结构为
```
pathMap = {
  '/home': RouteRecord,
  '/home/test': RouteRecord
  '/about': RouteRecord
}


RouteRecord 为各自route的内容
```
`pathList nameMap`一样的道理，就是递归循环遍历，得到里面扁平的内容，这样形成一个对象，我们就可以直接根据`path`直接拿到内容，这样`createRouteMap`完成了任务。

我们了解了`createRouteMap`之后，看看`createMatcher`的返回
```
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)

  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }

  function match() {
    ...
  }

  return {
    match,
    addRoutes
  }
}
```
我们`createRouteMap`的作用，也就是了解了`addRoutes`的作用，下面我们说说另一个返回`match`
```
export function createMatcher() {
  ...

  function match() {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      const record = nameMap[name]

      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }

      location.path = fillParams(record.path, location.params, `named route "${name}"`)

      return _createRoute(record, location, redirectedFrom)
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
}
```
里面我留了主要的代码，首先`const location = normalizeLocation(raw, currentRoute, false, router)`就是对传入的route做解析，解析出
```
return {
  _normalized: true,
  path,
  query,
  hash
}

里面解析法就不深入谈论
```
然后判断是否存在name，如果存在`name`执行`_createRoute()`，然后判断`path`，在执行`_createRoute()`，这么我们就知道跳转的方式是先根据`name`后根据`path`，这样我们就知道了 判断完毕之后主要是根据`_createRoute`函数做跳转
```
_createRoute

function _createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: Location
): Route {
  if (record && record.redirect) {
    return redirect(record, redirectedFrom || location)
  }
  if (record && record.matchAs) {
    return alias(record, location, record.matchAs)
  }
  return createRoute(record, location, redirectedFrom, router)
}
```
这里面就是看我们传递的参数是否存在`redirect`和`matchAs`
> 这里面的参数是处理之后的参数，源参数中不存在matchAs 这里的参数为const record: RouteRecord = {}
最后执行`createRoute`函数
```
createRoute

export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  const stringifyQuery = router && router.options.stringifyQuery

  let query: any = location.query || {}
  try {
    query = clone(query)
  } catch (e) {}

  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location, stringifyQuery),
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  return Object.freeze(route)
}
```
只要返回一个冻结的route。

**总结match**
主要是根据传入的`routes`处理参数，扁平化之后返回一个映射map`pathList pathMap nameMap`，然后根据跳转路由匹配`name`或者`path`，找到对应的组件。

![''](/assets/router/match.png)
上面是一张执行的堆栈图，首先执行`Vue.mixin`执行`this._router.init(this);`然后进入Route构造函数执行init，然后执行`history.transitionTo(history.getCurrentLocation());` 执行`transitionTo`会执行`var route = this.router.match(location, this.current);`，然后又执行
```
VueRouter.prototype.match = function match (
  raw,
  current,
  redirectedFrom
) {
  return this.matcher.match(raw, current, redirectedFrom)
};
```
有关matcher的match过程已经经过 就不多赘述了。也就是当我们访问一个路由的时候  就会执行match方法。

