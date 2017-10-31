# 第五章 辅助函数

在第一章我们曾经说过：

> VUEX采用的是典型的IIFE\(立即执行函数表达式\)模式，当代码被加载\(通过`<script>`或`Vue.use()`\)后，VUEX会返回一个对象，这个对象包含了`Store`类、`install`方法、`mapState`辅助函数、`mapMutations`辅助函数、`mapGetters`辅助函数、`mapActions`辅助函数、`createNamespacedHelpers`辅助函数以及当前的版本号`version`。

本章就将详细讲解`mapState`、`mapMutations`、`mapGetters`、`mapActions`、`createNamespacedHelpers这5个辅助和函数。`

## 5.1 主要辅助函数

### 5.1.1 mapState

如果你在使用VUEX过程中使用过`mapState`辅助函数将state映射为计算属性你应该会为它所支持的多样化的映射形式感到惊讶。我们不妨先来看看官方文档对它的介绍：

![](/assets/vuex2.PNG)

如果你深入思考过你可能会有疑问：VUEX的`mapState`是如何实现这么多种映射的呢？如果你现在还不明白，那么跟随我们来一起看看吧！

`mapState`辅助函数定义在VUEX源码中的790 ~ 815 行，主要是对多种映射方式以及带命名空间的模块提供了支持，我们来看看它的源码：

```
var mapState = normalizeNamespace(function (namespace, states) {
  var res = {};
  normalizeMap(states).forEach(function (ref) {
    var key = ref.key;
    var val = ref.val;

    res[key] = function mappedState () {
      var state = this.$store.state;
      var getters = this.$store.getters;
      if (namespace) {
        var module = getModuleByNamespace(this.$store, 'mapState', namespace);
        if (!module) {
          return
        }
        state = module.context.state;
        getters = module.context.getters;
      }
      return typeof val === 'function'
        ? val.call(this, state, getters)
        : state[val]
    };
    // mark vuex getter for devtools
    res[key].vuex = true;
  });
  return res
});
```

可以看到，mapState函数实际上是以函数表达式的形式的形式定义的，它的实际函数是normalizeNamespace函数，这个函数会对mapState函数的输入参数进行归一化/规范化处理，其最主要的功能是实现了支持带命名空间的模块，我们来看一下它的实现：

```
function normalizeNamespace (fn) {
  return function (namespace, map) {
    if (typeof namespace !== 'string') {
      map = namespace;
      namespace = '';
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/';
    }
    return fn(namespace, map)
  }
}
```

可以看到mapState实际上是中间的那段函数：

```
return function (namespace, map) {
  if (typeof namespace !== 'string') {
    map = namespace;
    namespace = '';
  } else if (namespace.charAt(namespace.length - 1) !== '/') {
    namespace += '/';
  }
  return fn(namespace, map)
}
```

它实际接收namespace, map可以接收两个参数，也可以只接受一个map参数。

1. 当用户只提供了一个map参数时。这种情况由于类型不是string，会直接将其作为map，并将namespace置为空字符串。这种情况其实就和官方文档的使用方式相匹配。我们可以看看官方文档的示例就是只提供了一个map参数，并没有提供namespace参数。
2. 当用户提供了namespace, map两个参数时，但namespace不是字符串类型。实际上这种情况会和情况1做一样的处理，传进去的第一个参数作为实际的map，第二个参数会被忽略。
3. 当用户提供了namespace, map两个参数时，且namespace是字符串类型。这种情况下会根据字符串最末一位字符串是否是反斜线\(/\)来区别对待。最终程序内部会将namespace统一处理成最后一位是反斜线\(/\)的字符串。

由以上分析我们可以知道，上述官方文档在此处的示例其实并不完善，该实例并没有指出**可以通过提供模块名称作为mapState的第一个参数来映射带命名空间的模块的state**。

我们举个例子看一下：

```
const moduleA = {
  namespaced: true,//带命名空间
  state: { count1: 1, age1: 20 }
}
const store = new Vuex.Store({
  state() {
    return {
        count: 0, age: 0
    }
  },
  modules: {
    a: moduleA
  }
})
var vm = new Vue({
  el: '#example',
  store,
  computed: Vuex.mapState('a', {// 映射时提供模块名作为第一个参数
     count1: state => state.count1,
     age1: state => state.age1,
  })
})
console.log(vm)
```

其输出如下：

![](/assets/vuex3.png)传递模块名称后，我们只能映射带命名空间的该模块的state，如果该模块不带命名空间（即没有设置namespace属性）、或者对于其它名字的模块，我们是不能映射他们的state的。

**传递了模块名称，但该模块不带命名空间，尝试对其进行映射：**

```
const moduleA = {
  // namespaced: true,
  state: { count1: 1, age1: 20 }
}
const store = new Vuex.Store({
  state() {
    return {
        count: 0, age: 0
    }
  },
  modules: {
    a: moduleA
  }
})
var vm = new Vue({
  el: '#example',
  store,
  computed: Vuex.mapState('a', {
     count1: state => state.count1,
     age1: state => state.age1,
  })
})
console.log(vm)
```

**传递了模块名称，但尝试映射其它模块的state：**

```
const moduleA = {
  namespaced: true,
  state: { count1: 1, age1: 20 }
}
const store = new Vuex.Store({
  state() {
    return {
        count: 0, age: 0
    }
  },
  modules: {
    a: moduleA
  }
})
var vm = new Vue({
  el: '#example',
  store,
  computed: Vuex.mapState('a', {
     count1: state => state.count,
     age1: state => state.age,
  })
})
console.log(vm)
```

这两种情况下的输出结果都会是undefined:

![](/assets/vuex5.png)

讲完了mapState的参数，我们接着回过头来看看mapState的实现。这里重复粘贴一下前面有关mapState定义的代码：

```
function normalizeNamespace (fn) {
  return function (namespace, map) {
    if (typeof namespace !== 'string') {
      map = namespace;
      namespace = '';
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/';
    }
    return fn(namespace, map)
  }
}
```

我们可以看到，在归一化/规范化输入参数后，mapState函数实际上是返回了另外一个函数的执行结果：

```
return fn(namespace, map)
```

这个`fn`就是以函数表达式定义mapState函数时的normalizeNamespace 函数的参数，我们在前面已经见到过。再次粘贴其代码以便于分析：

```
function (namespace, states) {
  var res = {};
  normalizeMap(states).forEach(function (ref) {
    var key = ref.key;
    var val = ref.val;

    res[key] = function mappedState () {
      var state = this.$store.state;
      var getters = this.$store.getters;
      if (namespace) {
        var module = getModuleByNamespace(this.$store, 'mapState', namespace);
        if (!module) {
          return
        }
        state = module.context.state;
        getters = module.context.getters;
      }
      return typeof val === 'function'
        ? val.call(this, state, getters)
        : state[val]
    };
    // mark vuex getter for devtools
    res[key].vuex = true;
  });
  return res
};
```

粗略来看，这个函数会重新定义map对象的key-value对，并作为一个新的对象返回。我们来进一步具体分析一下。

该函数首先调用normalizeMap函数对state参数进行归一化/规范化。normalizeMap函数定义在VUEX源码的899 ~ 903行，我们来具体看看它的实现：

```
function normalizeMap (map) {
  return Array.isArray(map)
    ? map.map(function (key) { return ({ key: key, val: key }); })
    : Object.keys(map).map(function (key) { return ({ key: key, val: map[key] }); })
}
```

该函数实际上意味着mapState函数的map参数同时支持数组和对象两种形式。

1. 如果是数组，则会遍历数组元素，将数组元素转成{value: value}对象。
2. 如果是对象，则会遍历对象key，以key-value构成{key: value}对象。

这两种形式最终都会得到一个新数组，而数组元素就是{key: value}形式的对象。

这也与官方文档的描述相印证，官方文档的既提供了mapState函数的map参数是对象的例子，也提供了参数是数组的例子。

回过头来看，normalizeMap\(states\)函数执行完后会遍历，针对每一个对象元素的value做进一步的处理。它首先拿的是根实例上挂载的store模块的state：

```
var state = this.$store.state;
var getters = this.$store.getters;
```

而如果mapState函数提供了命名空间参数（即模块名），则会拿带命名空间模块的state:

```
if (namespace) {
  var module = getModuleByNamespace(this.$store, 'mapState', namespace);
  if (!module) {
    return
  }
  state = module.context.state;
  getters = module.context.getters;
}
```

这其中会调用一个从根store开始，向下查找对应命名空间模块的方法getModuleByNamespace，它定义在VUEX源码的917 ~ 923 行：

```
function getModuleByNamespace (store, helper, namespace) {
  var module = store._modulesNamespaceMap[namespace];
  if ("development" !== 'production' && !module) {
    console.error(("[vuex] module namespace not found in " + helper + "(): " + namespace));
  }
  return module
}
```

因为我们在实例化Store类的时候已经把所有模块以namespace的为key的形式挂载在了根store实例的\_modulesNamespaceMap属性上，所以这个查询过程只是一个对象key的查找过程，实现起来比较简单。

回过头来继续看mapState函数中“`normalizeMap(states)函数执行完后会遍历，针对每一个对象元素的value做进一步的处理`”的最后的执行，它会根据原始的value是否是function而进一步处理：

1. 如果是不是function，则直接拿对应模块的state中key对应的value。
2. 如果是function，那么将会执行该function，并且会将state, getters分别暴露给该function作为第一个和第二个参数。

第二种情况在前述官方文档的例子中也有所体现：

    // 为了能够使用 `this` 获取局部状态，必须使用常规函数
    countPlusLocalState (state) {
      return state.count + this.localCount
    }

**但这个官方文档例子并不完整，它并没有体现出还会暴露出getters参数**，实际上，上述例子的完整形式应该是这样子的：

    // 为了能够使用 `this` 获取局部状态，必须使用常规函数
    countPlusLocalState (state, getters) {
      return state.count + this.localCount + getters.somegetter
    }

### 5.1.2 mapMutations

与mapState可以映射模块的state为计算属性类似，mapMutations也可以将模块的mutations映射为methods，我们来看看官方文档的介绍：

    import { mapMutations } from 'vuex'

    export default {
      // ...
      methods: {
        ...mapMutations([
          // 将 `this.increment()` 映射为 `this.$store.commit('increment')`
          'increment',

          // `mapMutations` 也支持载荷：
          // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
          'incrementBy' 
        ]),
        ...mapMutations({
          add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
        })
      }
    }

同样我们来看看它是如何实现的，它的实现定义在VUEX源码中的817 ~ 841 行：

```
var mapMutations = normalizeNamespace(function (namespace, mutations) {
  var res = {};
  normalizeMap(mutations).forEach(function (ref) {
    var key = ref.key;
    var val = ref.val;

    res[key] = function mappedMutation () {
      var args = [], len = arguments.length;
      while ( len-- ) args[ len ] = arguments[ len ];

      var commit = this.$store.commit;
      if (namespace) {
        var module = getModuleByNamespace(this.$store, 'mapMutations', namespace);
        if (!module) {
          return
        }
        commit = module.context.commit;
      }
      return typeof val === 'function'
        ? val.apply(this, [commit].concat(args))
        : commit.apply(this.$store, [val].concat(args))
    };
  });
  return res
});
```

和mapState的实现几乎完全一样，唯一的差别只有两点：

1. 提交mutaion时可以传递载荷，所以这里有一步是拷贝载荷。
2. mutation是用来提交的，所以这里拿的是commit。

我们来具体分析一下代码的执行：

首先是拷贝载荷：

```
var args = [], len = arguments.length;
while ( len-- ) args[ len ] = arguments[ len ];
```

然后是拿commit，如果mapMutations函数提供了命名空间参数（即模块名），则会拿带命名空间模块的commit:

```
var commit = this.$store.commit;
if (namespace) {
  var module = getModuleByNamespace(this.$store, 'mapMutations', namespace);
  if (!module) {
    return
  }
  commit = module.context.commit;
}
```

最后则会看对应mutation的value是不是函数：

1. 如果不是函数，则直接执行commit，参数是value和载荷组成的数组。
2. 如果是函数，则直接执行该函数，并将comit作为其第一个参数，arg仍然作为后续参数。

也就是说，**官方文档例子并不完整，它并没有体现第二种情况**，实际上，官方文档例子的完整形式还应当包括：

    import { mapMutations } from 'vuex'

    export default {
      // ...
      methods: {
        ...mapMutations('moduleName', {
           addAlias: function(commit, playload) {
               //将 `this.addAlias()` 映射为 `this.$store.commit('increment', amount)`
               commit('increment') 
               //将 `this.addAlias(playload)` 映射为 `this.$store.commit('increment', playload)`
               commit('increment', playload)
           }
         })
      }
    }

同样，mapMutations上述映射方式都支持传递一个模块名作为命名空间参数，这个在官方文档也没有体现：

    import { mapMutations } from 'vuex'

    export default {
      // ...
      methods: {
        ...mapMutations('moduleName', [
          // 将 `this.increment()` 映射为 `this.$store.commit('increment')`
          'increment',

          // `mapMutations` 也支持载荷：
          // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
          'incrementBy' 
        ]),
        ...mapMutations('moduleName', {
          // 将 `this.add()` 映射为 `this.$store.commit('increment')`
          add: 'increment' 
        }),
        ...mapMutations('moduleName', {
          addAlias: function(commit) {
              //将 `this.addAlias()` 映射为 `this.$store.commit('increment')`
              commit('increment') 
          }
        })
      }
    }

我们可以举个例子证明一下：

    const moduleA = {
      namespaced: true,
      state: { source: 'moduleA' },
      mutations: {
        increment (state, playload) {
          // 这里的 `state` 对象是模块的局部状态
          state.source += playload
        }
      }
    }
    const store = new Vuex.Store({
      state() {
        return {
            source: 'root'
        }
      },
      mutations: {
        increment (state, playload) {
          state.source += playload
        }
      },
      modules: {
        a: moduleA
      }
    })
    var vm = new Vue({
      el: '#example',
      store,
      mounted() {
        console.log(this.source)
        this.localeincrement('testdata')
        console.log(this.source)
      },
        computed: Vuex.mapState([
          'source'
        ]
      ),
      methods: {
        ...Vuex.mapMutations({
          localeincrement (commit, args) {
            commit('increment', args)
          }
        })
      }
    })

输出结果：

```
root
test.html:139 roottestdata
```

另外一个例子：

    const moduleA = {
      namespaced: true,
      state: { source: 'moduleA' },
      mutations: {
        increment (state, playload) {
          // 这里的 `state` 对象是模块的局部状态
          state.source += playload
        }
      }
    }
    const store = new Vuex.Store({
      state() {
        return {
            source: 'root'
        }
      },
      mutations: {
        increment (state, playload) {
          state.source += playload
        }
      },
      modules: {
        a: moduleA
      }
    })
    var vm = new Vue({
      el: '#example',
      store,
      mounted() {
        console.log(this.source)
        this.localeincrement('testdata')
        console.log(this.source)
      },
        computed: Vuex.mapState('a', [
          'source'
        ]
      ),
      methods: {
        ...Vuex.mapMutations('a', {
          localeincrement (commit, args) {
            commit('increment', args)
          }
        })
      }
    })

输入结果：

```
moduleA
test.html:139 moduleAtestdata
```

### 5.1.3 mapGetters

与mapState可以映射模块的state为计算属性类似，mapGetters也可以将模块的getters映射为计算属性，我们来看看官方文档的介绍：

![](/assets/vuex6.png)

mapGetters辅助函数定义在VUEX源码中的843 ~ 864 行，我们来看看它的源码：

```
var mapGetters = normalizeNamespace(function (namespace, getters) {
  var res = {};
  normalizeMap(getters).forEach(function (ref) {
    var key = ref.key;
    var val = ref.val;

    val = namespace + val;
    res[key] = function mappedGetter () {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      if ("development" !== 'production' && !(val in this.$store.getters)) {
        console.error(("[vuex] unknown getter: " + val));
        return
      }
      return this.$store.getters[val]
    };
    // mark vuex getter for devtools
    res[key].vuex = true;
  });
  return res
});
```

和mapState的实现几乎完全一样，唯一的差别只有1点：就是最后不会出现value为函数的情况。直接拿的是对应模块上的getters:

```
return this.$store.getters[val]
```

### 5.1.4 mapActions

与mapMutations可以映射模块的mutation为methods类似，mapActions也可以将模块的actions映射为methods，我们来看看官方文档的介绍：

    import { mapActions } from 'vuex'

    export default {
      // ...
      methods: {
        ...mapActions([
          // 将 `this.increment()` 映射为 `this.$store.dispatch('increment')`
          'increment',

          // `mapActions` 也支持载荷：
          // 将 `this.incrementBy(amount)` 映射为 `this.$store.dispatch('incrementBy', amount)`
          'incrementBy' 
        ]),
        ...mapActions({
          // 将 `this.add()` 映射为 `this.$store.dispatch('increment')`
          add: 'increment' 
        })
      }
    }

同样我们来看看它是如何实现的，它的实现定义在VUEX源码中的866 ~ 890 行：

```
var mapActions = normalizeNamespace(function (namespace, actions) {
  var res = {};
  normalizeMap(actions).forEach(function (ref) {
    var key = ref.key;
    var val = ref.val;

    res[key] = function mappedAction () {
      var args = [], len = arguments.length;
      while ( len-- ) args[ len ] = arguments[ len ];

      var dispatch = this.$store.dispatch;
      if (namespace) {
        var module = getModuleByNamespace(this.$store, 'mapActions', namespace);
        if (!module) {
          return
        }
        dispatch = module.context.dispatch;
      }
      return typeof val === 'function'
        ? val.apply(this, [dispatch].concat(args))
        : dispatch.apply(this.$store, [val].concat(args))
    };
  });
  return res
});
```

和mapMutations的实现几乎完全一样，唯一的差别只有1点：

1. action是用来分派的，所以这里拿的是dispatch。

我们来具体分析一下代码的执行：

首先是拷贝载荷：

```
var args = [], len = arguments.length;
while ( len-- ) args[ len ] = arguments[ len ];
```

然后是拿dispatch，如果mapActions函数提供了命名空间参数（即模块名），则会拿带命名空间模块的dispatch:

```
var dispatch = this.$store.dispatch;
if (namespace) {
  var module = getModuleByNamespace(this.$store, 'mapActions', namespace);
  if (!module) {
    return
  }
  dispatch = module.context.dispatch;
}
```

最后则会看对应action的value是不是函数：

1. 如果不是函数，则直接执行dispatch，参数是value和载荷组成的数组。
2. 如果是函数，则直接执行该函数，并将dispatch作为其第一个参数，arg仍然作为后续参数。

也就是说，**官方文档例子并不完整，它并没有体现第二种情况**，实际上，官方文档例子的完整形式还应当包括：

    import { mapActions } from 'vuex'

    export default {
      // ...
      methods: {
        ...mapActions ('moduleName', {
           addAlias: function(dispatch, playload) {
               //将 `this.addAlias()` 映射为 `this.$store.dispatch('increment', amount)`
               dispatch('increment') 
               //将 `this.addAlias(playload)` 映射为 `this.$store.dispatch('increment', playload)`
               dispatch('increment', playload)
           }
         })
      }
    }

同样，mapActions上述映射方式都支持传递一个模块名作为命名空间参数，这个在官方文档也没有体现：

    import { mapActions } from 'vuex'

    export default {
      // ...
      methods: {
        ...mapActions('moduleName', [
          // 将 `this.increment()` 映射为 `this.$store.dispatch('increment')`
          'increment', 

          // `mapActions` 也支持载荷：
          // 将 `this.incrementBy(amount)` 映射为 `this.$store.dispatch('incrementBy', amount)`
          'incrementBy' 
        ]),
        ...mapActions('moduleName', {
          // 将 `this.add()` 映射为 `this.$store.dispatch('increment')`
          add: 'increment' 
        }),
        ...mapActions('moduleName', {
          addAlias: function (dispatch) {
            // 将 `this.addAlias()` 映射为 `this.$store.dispatch('increment')`
            dispatch('increment') 
          }
        })
      }
    }

我们可以举个例子证明一下：

    const moduleA = {
      namespaced: true,
      state: { source: 'moduleA' },
      mutations: {
        increment (state, playload) {
          // 这里的 `state` 对象是模块的局部状态
          state.source += playload
        }
      },
      actions: {
        increment (context) {
          context.commit('increment', 'testdata')
        }
      }
    }
    const store = new Vuex.Store({
      state() {
        return {
            source: 'root'
        }
      },
      mutations: {
        increment (state, playload) {
          state.source += playload
        }
      },
      actions: {
        increment (context) {
          context.commit('increment', 'testdata')
        }
      },
      modules: {
        a: moduleA
      }
    })
    var vm = new Vue({
      el: '#example',
      store,
      mounted() {
        console.log(this.source)
        this.localeincrement()
        console.log(this.source)
      },
        computed: Vuex.mapState([
          'source'
        ]
      ),
      methods: {
        ...Vuex.mapActions( {
          localeincrement (dispatch) {
            dispatch('increment')
          }
        })
      }
    })

输出结果：

```
root
roottestdata
```

另外一个例子：

    const moduleA = {
      namespaced: true,
      state: { source: 'moduleA' },
      mutations: {
        increment (state, playload) {
          // 这里的 `state` 对象是模块的局部状态
          state.source += playload
        }
      },
      actions: {
        increment (context) {
          context.commit('increment', 'testdata')
        }
      }
    }
    const store = new Vuex.Store({
      state() {
        return {
            source: 'root'
        }
      },
      mutations: {
        increment (state, playload) {
          state.source += playload
        }
      },
      actions: {
        increment (context) {
          context.commit('increment', 'testdata')
        }
      },
      modules: {
        a: moduleA
      }
    })
    var vm = new Vue({
      el: '#example',
      store,
      mounted() {
        console.log(this.source)
        this.localeincrement()
        console.log(this.source)
      },
        computed: Vuex.mapState('a', [
          'source'
        ]
      ),
      methods: {
        ...Vuex.mapActions('a', {
          localeincrement (dispatch) {
            dispatch('increment')
          }
        })
      }
    })

输出结果：

```
moduleA
moduleAtestdata
```

### 5.1.5 createNamespacedHelpers

createNamespacedHelpers主要是根据传递的命名空间产生对应模块的局部化mapState、mapGetters、mapMutations、mapActions映射函数，它定义在VUEX源码的892 ~ 897行：

```
var createNamespacedHelpers = function (namespace) { return ({
  mapState: mapState.bind(null, namespace),
  mapGetters: mapGetters.bind(null, namespace),
  mapMutations: mapMutations.bind(null, namespace),
  mapActions: mapActions.bind(null, namespace)
}); };
```

## 5.2 其它辅助函数

### 5.2.1 isObject

isObject定义在VUEX源码的94 ~ 96 行，主要判断目标是否是有效对象，其实现比较简单：

```
//判断是不是object
function isObject (obj) {
  return obj !== null && typeof obj === 'object'
}
```

### 5.2.2 isPromise

isPromise定义在VUEX源码的98 ~ 100 行，主要判断目标是否是promise，其实现比较简单：

```
function isPromise (val) {
  return val && typeof val.then === 'function'
}
```

### 5.2.3 assert

assert定义在VUEX源码的102 ~ 104 行，主要用来断言，其实现比较简单：

```
function assert (condition, msg) {
  if (!condition) { throw new Error(("[vuex] " + msg)) }
}
```



