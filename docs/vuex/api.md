## vuex源码分析之API

我们知道vuex的常用API为`state`， `getters`， `mutations`，`actions`;

### 数据获取

#### state
我们从store的实例化中知道，其实实例化就是把父module和子module根据namespace去扁平化，存储到私有变量中`_action`，`_mutitation`，`state`，`getter`中

我们常常调用的方式为
```
this.$store.state.xxxx
```
我们就知道是找this.$store也就是store实例化的state方法。

```
export class Store {
  ...

  get state () {
    return this._vm._data.$$state
  }

  ...
}

function resetStoreVM (store, state, hot) {
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
}
```

上面两段代码我们知道 当我们访问`get state () {}`的时候就是拿store对象上的`state`。

#### getter
在store对象中我们讲过
```
function resetStoreVM (store, state, hot) {
  // ...
  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  // ...
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  // ...
}
```
在`resetStoreVM`过程中的时候，`store.getters`被监听了，当被访问的时候返回`store._vm[key]`，而`store._vm[key]`其实是`store._vm = new Vue({`的`computed`属性，`computed`又是由`wrappedGetters`遍历得到的`computed[key] = () => fn(store)`函数集合，`wrappedGetters`是我们定义getter的集合。
```
wrappedGetters:


const store = new Vuex.Store({
  modules: {
    a: {
      getters: {
        getterA(state) {
          return state.root + "getter";
        },
      },
    },
    b: {
      getters: {
        getterB(state) {
          return state.root + "getter";
        },
      },
    },
  },
  ...
  getters: {
    getterRoot(state) {
      return state.root + "getter";
    },
  },
});

为集合所有getter的函数，收集方法在store已经讲了。
```

### 数据存储
#### mutation

我们修改数据的时候使用的方式为`this.$store.commit('a', 1)`

```
commit (_type, _payload, _options) {
  // check object-style commit
  const {
    type,
    payload,
    options
  } = unifyObjectStyle(_type, _payload, _options)

  const mutation = { type, payload }
  const entry = this._mutations[type]
  if (!entry) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] unknown mutation type: ${type}`)
    }
    return
  }
  this._withCommit(() => {
    entry.forEach(function commitIterator (handler) {
      handler(payload)
    })
  })
  this._subscribers.forEach(sub => sub(mutation, this.state))

  if (
    process.env.NODE_ENV !== 'production' &&
    options && options.silent
  ) {
    console.warn(
      `[vuex] mutation type: ${type}. Silent option has been removed. ` +
      'Use the filter functionality in the vue-devtools'
    )
  }
}
```
源码中其实就是在之前收集的`this._mutations[type]`中去找type
```
this._mutations的内容为

this._mutations = {
  mutationsRoot: fn...
  'a/mutationsA': fn...
  'mutationsB': fn...
}

fn为我们new Vuex.Store时传的mutation函数
```
找到对象的type执行对应的函数。


#### action
```
dispatch (_type, _payload) {
  // check object-style dispatch
  const {
    type,
    payload
  } = unifyObjectStyle(_type, _payload)

  const action = { type, payload }
  const entry = this._actions[type]
  if (!entry) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] unknown action type: ${type}`)
    }
    return
  }

  this._actionSubscribers.forEach(sub => sub(action, this.state))

  return entry.length > 1
    ? Promise.all(entry.map(handler => handler(payload)))
    : entry[0](payload)
}
```
从源码中我们可以看出其实和commit类似，只不过`this._actions[type]`存储的方式为
```
this._actions = {
  actionRoot: [fn,fn,...],
  'a/actionA': [fn, fn, ...]
}

fn为我们new Vuex.Store时传的action函数
```
遍历数组中的函数，全部执行后返回，因为dispatch是可以重名的。
