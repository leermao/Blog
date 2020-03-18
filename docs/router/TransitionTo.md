## Vue-router源码分析之路径转换

### 导航守卫
我们首先了解一下页面跳转的方法
```
初始话的时候
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

手动执行this.$router.push / this.$router.replace
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
```
执行了自己的push/replace方法
```
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const { current: fromRoute } = this
  this.transitionTo(
    location,
    route => {
      pushHash(route.fullPath)
      handleScroll(this.router, route, fromRoute, false)
      onComplete && onComplete(route)
    },
    onAbort
  )
}

replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const { current: fromRoute } = this
  this.transitionTo(
    location,
    route => {
      replaceHash(route.fullPath)
      handleScroll(this.router, route, fromRoute, false)
      onComplete && onComplete(route)
    },
    onAbort
  )
}
```
在history中我们了解也是执行的`transitionTo`方法，所以我们知道跳转的逻辑就是在`transitionTo`中

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
```
在父类Histroy中
```
transitionTo () {
  const route = this.router.match(location, this.current)
  this.confirmTransition(
    route,
    () => {
      this.updateRoute(route)
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => {
          cb(route)
        })
      }
    },
    err => {
      if (onAbort) {
        onAbort(err)
      }
      if (err && !this.ready) {
        this.ready = true
        this.readyErrorCbs.forEach(cb => {
          cb(err)
        })
      }
    }
  )
}

匹配出当前route，传入到confirmTransition

confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
  const current = this.current
  const abort = err => {
    ...处理异常
  }
  if (
    isSameRoute(route, current) &&
    // in the case the route map has been dynamically appended to
    route.matched.length === current.matched.length
  ) {
    ... 处理相同路由
  }

  const { updated, deactivated, activated } = resolveQueue(
    this.current.matched,
    route.matched
  )

  const queue: Array<?NavigationGuard> = [].concat(
    // in-component leave guards
    extractLeaveGuards(deactivated),
    // global before hooks
    this.router.beforeHooks,
    // in-component update hooks
    extractUpdateHooks(updated),
    // in-config enter guards
    activated.map(m => m.beforeEnter),
    // async components
    resolveAsyncComponents(activated)
  )

  this.pending = route
  const iterator = (hook: NavigationGuard, next) => {
    if (this.pending !== route) {
      return abort()
    }
    try {
      hook(route, current, (to: any) => {
        if (to === false || isError(to)) {
          // next(false) -> abort navigation, ensure current URL
          this.ensureURL(true)
          abort(to)
        } else if (
          typeof to === 'string' ||
          (typeof to === 'object' &&
            (typeof to.path === 'string' || typeof to.name === 'string'))
        ) {
          // next('/') or next({ path: '/' }) -> redirect
          abort()
          if (typeof to === 'object' && to.replace) {
            this.replace(to)
          } else {
            this.push(to)
          }
        } else {
          // confirm transition and pass on the value
          next(to)
        }
      })
    } catch (e) {
      abort(e)
    }
  }

  runQueue(queue, iterator, () => {
    const postEnterCbs = []
    const isValid = () => this.current === route
    // wait until async components are resolved before
    // extracting in-component enter guards
    const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
    const queue = enterGuards.concat(this.router.resolveHooks)
    runQueue(queue, iterator, () => {
      if (this.pending !== route) {
        return abort()
      }
      this.pending = null
      onComplete(route)
      if (this.router.app) {
        this.router.app.$nextTick(() => {
          postEnterCbs.forEach(cb => {
            cb()
          })
        })
      }
    })
  })
}
```
上面我们看`transitionTo`中开始获取route，然后传入处理函数，刚开始判断一些异常情况，接着就是重点，就是执行路由钩子。

![](/assets/router/hook.png)
这个就是整个hook的执行顺序，下面我们一点点解析。

