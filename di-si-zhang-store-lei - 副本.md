# 第四章 Store类

Store类定义在VUEX源码的294 ~ 775 行，是VUEX中最后定义的、最重要的类，也是我们实际上最后使用的类。我们来具体看看它的内容。

## 4.1 成员属性和成员函数

Store类的成员属性主要有下面几个：

1. \_committing。标识是否正提交。类型Boolean.
2. \_actions。储存actions。类型Object。
3. \_actionSubscribers。存储actions的订阅者。类型Array。
4. \_mutations。存储mutations。类型Object。
5. \_wrappedGetters。存储wrapped后的Getters。类型Object。
6. \_modules。存储模块链。是一个ModuleCollection实例。
7. \_modulesNamespaceMap。存储带命名空间的modules。类型Object。
8. \_subscribers。存储订阅者。类型Object。
9. \_watcherVM。存储Vue实例。
10. strict。标识是否strict模式。类型Boolean。

其源码定义在296 ~ 362 Store类的构造函数中，我们来看看：

    var Store = function Store (options) {
      var this$1 = this;
      if ( options === void 0 ) options = {};

      // Auto install if it is not done yet and `window` has `Vue`.
      // To allow users to avoid auto-installation in some cases,
      // this code should be placed here. See #731
      if (!Vue && typeof window !== 'undefined' && window.Vue) {
        install(window.Vue);
      }

      {
        assert(Vue, "must call Vue.use(Vuex) before creating a store instance.");
        assert(typeof Promise !== 'undefined', "vuex requires a Promise polyfill in this browser.");
        assert(this instanceof Store, "Store must be called with the new operator.");
      }

      var plugins = options.plugins; if ( plugins === void 0 ) plugins = [];
      var strict = options.strict; if ( strict === void 0 ) strict = false;

      var state = options.state; if ( state === void 0 ) state = {};
      if (typeof state === 'function') {
        state = state() || {};
      }

      // store internal state
      this._committing = false;
      this._actions = Object.create(null);
      this._actionSubscribers = [];
      this._mutations = Object.create(null);
      this._wrappedGetters = Object.create(null);
      this._modules = new ModuleCollection(options);
      this._modulesNamespaceMap = Object.create(null);
      this._subscribers = [];
      this._watcherVM = new Vue();

      // bind commit and dispatch to self
      var store = this;
      var ref = this;
      var dispatch = ref.dispatch;
      var commit = ref.commit;
      this.dispatch = function boundDispatch (type, payload) {
        return dispatch.call(store, type, payload)
      };
      this.commit = function boundCommit (type, payload, options) {
        return commit.call(store, type, payload, options)
      };

      // strict mode
      this.strict = strict;

      // init root module.
      // this also recursively registers all sub-modules
      // and collects all module getters inside this._wrappedGetters
      installModule(this, state, [], this._modules.root);

      // initialize the store vm, which is responsible for the reactivity
      // (also registers _wrappedGetters as computed properties)
      resetStoreVM(this, state);

      // apply plugins
      plugins.forEach(function (plugin) { return plugin(this$1); });

      if (Vue.config.devtools) {
        devtoolPlugin(this);
      }
    };

该类会首先检查我们在声明Store实例的时候有没有传递options参数，如果没有则会初始化为空对象{}。然后会检查Vue有没有定义，如果Vue没有定义，并且window上已经有挂载Vue，那么会安装Vue，否则告警：

    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue);
    }

    {
      assert(Vue, "must call Vue.use(Vuex) before creating a store instance.");
      assert(typeof Promise !== 'undefined', "vuex requires a Promise polyfill in this browser.");
      assert(this instanceof Store, "Store must be called with the new operator.");
    }

这段代码可以确保Vue被且仅被一次安装。Vue是全局声明在VUEX源码中的294行：


	var Vue; // bind on install


而如果我们在页面使用了Vue（不管是脚本引入&lt;script src="./vue.js"&gt;&lt;/script&gt;还是node引入模式），在window上都会挂载一个Vue：

![](/assets/vuex8.png)

而安装Vue的install方法定义在VUEX源码的777 ~ 788行，主要就是给全局声明的Vue赋值，然后调用了applyMixin执行混入：


	function install (_Vue) {
	  if (Vue && _Vue === Vue) {
	    {
	      console.error(
	        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
	      );
	    }
	    return
	  }
	  Vue = _Vue;
	  applyMixin(Vue);
	}


applyMixin定义在VUEX源码的12 ~ 29行，它的主要目的是确保在Vue的beforeCreate钩子函数中调用vuexInit函数，当然更具Vue版本差异实现方法也有差异，因为我们主要针对&gt;2的版本，所以这里只看版本&gt;2时的情况：


	var applyMixin = function (Vue) {
	  var version = Number(Vue.version.split('.')[0]);
	
	  if (version >= 2) {
	    Vue.mixin({ beforeCreate: vuexInit });
	  } else {
	    // override init and inject vuex init procedure
	    // for 1.x backwards compatibility.
	    var _init = Vue.prototype._init;
	    Vue.prototype._init = function (options) {
	      if ( options === void 0 ) options = {};
	
	      options.init = options.init
	        ? [vuexInit].concat(options.init)
	        : vuexInit;
	      _init.call(this, options);
	    };
	  }
	
	  /**
	   * Vuex init hook, injected into each instances init hooks list.
	   */
	
	  function vuexInit () {
	    var options = this.$options;
	    // store injection
	    if (options.store) {
	      this.$store = typeof options.store === 'function'
	        ? options.store()
	        : options.store;
	    } else if (options.parent && options.parent.$store) {
	      this.$store = options.parent.$store;
	    }
	  }
	};


在该函数内部定义了vuexInit函数，该函数的主要作用是在Vue的实例上挂载$store：

![](/assets/vuex9.png)回过头来继续看Store构造函数，在执行完install之后，它会对构造函数的options参数进行检查，当不合法时会给出默认值：


	var plugins = options.plugins; if ( plugins === void 0 ) plugins = [];
	var strict = options.strict; if ( strict === void 0 ) strict = false;
	
	var state = options.state; if ( state === void 0 ) state = {};
	if (typeof state === 'function') {
	  state = state() || {};
	}


这里面有两点需要注意：

1. 判断undefiend采用的是void 0形式判断，这是一种非常好的判断方式。
2. state属性并不一定需要是个对象，它也可以是产生对象的工厂函数。这个我们在第二章Module类中已经分析过。

接下来就是成员属性的定义：


	// store internal state
	  this._committing = false;
	  this._actions = Object.create(null);
	  this._actionSubscribers = [];
	  this._mutations = Object.create(null);
	  this._wrappedGetters = Object.create(null);
	  this._modules = new ModuleCollection(options);
	  this._modulesNamespaceMap = Object.create(null);
	  this._subscribers = [];
	  this._watcherVM = new Vue();


再接下来就是成员函数dispatch和commit的定义，这个我们在下一节详细讲述。

接下来会执行installModule函数：

	// init root module.
	// this also recursively registers all sub-modules
	// and collects all module getters inside this._wrappedGetters
	installModule(this, state, [], this._modules.root);


英文的注释已经描述的很详细了，installModule会初始化根模块，并递归地注册子模块，收集所有模块的Getters放在\_wrappedGetters属性中。installModule是一个全局函数，定义在VUEX源码的577 ~ 617 行：


	function installModule (store, rootState, path, module, hot) {
	  var isRoot = !path.length;
	  var namespace = store._modules.getNamespace(path);
	
	  // register in namespace map
	  if (module.namespaced) {
	    store._modulesNamespaceMap[namespace] = module;
	  }
	
	  // set state
	  if (!isRoot && !hot) {
	    var parentState = getNestedState(rootState, path.slice(0, -1));
	    var moduleName = path[path.length - 1];
	    store._withCommit(function () {
	      Vue.set(parentState, moduleName, module.state);
	    });
	  }
	
	  var local = module.context = makeLocalContext(store, namespace, path);
	
	  module.forEachMutation(function (mutation, key) {
	    var namespacedType = namespace + key;
	    registerMutation(store, namespacedType, mutation, local);
	  });
	
	  module.forEachAction(function (action, key) {
	    var type = action.root ? key : namespace + key;
	    var handler = action.handler || action;
	    registerAction(store, type, handler, local);
	  });
	
	  module.forEachGetter(function (getter, key) {
	    var namespacedType = namespace + key;
	    registerGetter(store, namespacedType, getter, local);
	  });
	
	  module.forEachChild(function (child, key) {
	    installModule(store, rootState, path.concat(key), child, hot);
	  });
	}


代码开始会判断是否是根模块，并且会获取模块对应的命名空间，如果该模块有namespace属性，则在\_modulesNamespaceMap属性上以命名空间为key保存该模块：


	var isRoot = !path.length;
	var namespace = store._modules.getNamespace(path);
	
	// register in namespace map
	if (module.namespaced) {
	  store._modulesNamespaceMap[namespace] = module;
	}


下面这个貌似意思是将该模块的state以命名空间为key挂载父模块的state上，形成state链，这个过程是为了后面使用getNestedState函数查找对应命名空间的state：


	// set state
	if (!isRoot && !hot) {
	  var parentState = getNestedState(rootState, path.slice(0, -1));
	  var moduleName = path[path.length - 1];
	  store._withCommit(function () {
	    Vue.set(parentState, moduleName, module.state);
	  });
	}


我们来看一个根store上挂载namespaced的a模块，a模块又挂载namespaced的c模块的例子，此时根store的state长这样：

![](/assets/vuex11.png)

回过头来看installModule，接下来是拿到本地的上下文：


	var local = module.context = makeLocalContext(store, namespace, path);


本地上下文是什么意思呢？意思就是说，dispatch, commit, state, getters都是局部化了。我们知道store的模块是有命名空间的概念的，要想操作某一层级的东西，都是需要用命名空间去指定的。如果不使用命名空间，操作的是根级别的。所以本地上下文是指的就是某个模块的上下文，当你操作dispatch, commit, state, getters等的时候，你实际上直接操作的某个模块。

这个看似复杂的东西是如何实现的呢？其实它只不过是个语法糖，它内部也是通过逐级查找，找到对应的模块完成的。我们来看看这个makeLocalContext的实现，它定义在VUEX源码的618 ~ 675 行：

	
	/**
	 * make localized dispatch, commit, getters and state
	 * if there is no namespace, just use root ones
	 */
	function makeLocalContext (store, namespace, path) {
	  var noNamespace = namespace === '';
	
	  var local = {
	    dispatch: noNamespace ? store.dispatch : function (_type, _payload, _options) {
	      var args = unifyObjectStyle(_type, _payload, _options);
	      var payload = args.payload;
	      var options = args.options;
	      var type = args.type;
	
	      if (!options || !options.root) {
	        type = namespace + type;
	        if ("development" !== 'production' && !store._actions[type]) {
	          console.error(("[vuex] unknown local action type: " + (args.type) + ", global type: " + type));
	          return
	        }
	      }
	
	      return store.dispatch(type, payload)
	    },
	
	    commit: noNamespace ? store.commit : function (_type, _payload, _options) {
	      var args = unifyObjectStyle(_type, _payload, _options);
	      var payload = args.payload;
	      var options = args.options;
	      var type = args.type;
	
	      if (!options || !options.root) {
	        type = namespace + type;
	        if ("development" !== 'production' && !store._mutations[type]) {
	          console.error(("[vuex] unknown local mutation type: " + (args.type) + ", global type: " + type));
	          return
	        }
	      }
	
	      store.commit(type, payload, options);
	    }
	  };
	
	  // getters and state object must be gotten lazily
	  // because they will be changed by vm update
	  Object.defineProperties(local, {
	    getters: {
	      get: noNamespace
	        ? function () { return store.getters; }
	        : function () { return makeLocalGetters(store, namespace); }
	    },
	    state: {
	      get: function () { return getNestedState(store.state, path); }
	    }
	  });
	
	  return local
	}


该函接受根级别的store、命名空间、模块路径作为参数，然后声明一个local对象，分别局部化dispatch, commit, getters , state。我们来分别看一下：

**局部化dispatch:**

dispatch局部化时，首先判断有没有命名空间，如果没有则直接使用根级别的dispatch方法，如果有，则重新定义该方法:


	function (_type, _payload, _options) {
	  var args = unifyObjectStyle(_type, _payload, _options);
	  var payload = args.payload;
	  var options = args.options;
	  var type = args.type;
	
	  if (!options || !options.root) {
	    type = namespace + type;
	    if ("development" !== 'production' && !store._actions[type]) {
	      console.error(("[vuex] unknown local action type: " + (args.type) + ", global type: " + type));
	      return
	    }
	  }
	
	  return store.dispatch(type, payload)
	}


重新定义其实只不过是规范化参数，将参数映射到指定命名空间的模块上，是调用unifyObjectStyle来完成的，它定义在VUEX源码的763 ~ 775 行，我们来看看它的实现：

	function unifyObjectStyle (type, payload, options) {
	  if (isObject(type) && type.type) {
	    options = payload;
	    payload = type;
	    type = type.type;
	  }
	
	  {
	    assert(typeof type === 'string', ("Expects string as the type, but found " + (typeof type) + "."));
	  }
	
	  return { type: type, payload: payload, options: options }
	}


规范化参数其实是和官方文档关于dispatch的描述是呼应的，我们引用一下官方的表述：

> Actions 支持同样的载荷方式和对象方式进行分发：
> 
	 // 以载荷形式分发
	 store.dispatch('incrementAsync', {
	   amount: 10
	 })
>	
	 // 以对象形式分发
	 store.dispatch({
	   type: 'incrementAsync',
	   amount: 10
	 })
>

可以看出unifyObjectStyle只不过是把载荷形式的分发转换成了对象形式的分发，这种情况下的options其实是undefined。

回过头来继续看dispatch的局部化过程。当规范化参数会分别拿到规范化的参数，然后对于非根级别，则给type加上命名空间：


	var payload = args.payload;
	var options = args.options;
	var type = args.type;
	
	if (!options || !options.root) {
	  type = namespace + type;
	  if ("development" !== 'production' && !store._actions[type]) {
	    console.error(("[vuex] unknown local action type: " + (args.type) + ", global type: " + type));
	    return
	  }
	}


最后还是调用根级别的dispatch来完成分发：


	return store.dispatch(type, payload)


**局部化commit:**

commit的局部化如下：


	commit: noNamespace ? store.commit : function (_type, _payload, _options) {
	    var args = unifyObjectStyle(_type, _payload, _options);
	    var payload = args.payload;
	    var options = args.options;
	    var type = args.type;
	
	    if (!options || !options.root) {
	      type = namespace + type;
	      if ("development" !== 'production' && !store._mutations[type]) {
	        console.error(("[vuex] unknown local mutation type: " + (args.type) + ", global type: " + type));
	        return
	      }
	    }
	
	    store.commit(type, payload, options);
	  }
	};


整个过程和dispatch的局部化过程几乎一模一样，这里我们就不再赘述。

**局部化Getters:**

Getters的局部化时在前述的local对象上定义getters属性，并重新定义该属性的get函数：


	getters: {
	  get: noNamespace
	    ? function () { return store.getters; }
	    : function () { return makeLocalGetters(store, namespace); }
	},


其主要思路就是：没有命名空间的时候直接拿根级别的getters，有命名空间的时候拿对应模块上的getters。这其中用到了makeLocalGetters函数，它定义在VUEX源码的677 ~ 698 行，我们来看看它的实现：


	function makeLocalGetters (store, namespace) {
	  var gettersProxy = {};
	
	  var splitPos = namespace.length;
	  Object.keys(store.getters).forEach(function (type) {
	    // skip if the target getter is not match this namespace
	    if (type.slice(0, splitPos) !== namespace) { return }
	
	    // extract local getter type
	    var localType = type.slice(splitPos);
	
	    // Add a port to the getters proxy.
	    // Define as getter property because
	    // we do not want to evaluate the getters in this time.
	    Object.defineProperty(gettersProxy, localType, {
	      get: function () { return store.getters[type]; },
	      enumerable: true
	    });
	  });
	
	  return gettersProxy
	}


我们只需要遍历根级别store的getters属性，找到对应的命名空间，然后代理对它的访问就可以了。我们可以先来看一个例子以及根级别上getters的内容，相信对理解上述代码会更用帮助：

    const moduleC = {
      namespaced: true,
      state: { count: 1, age1: 20 },
      mutations: {
        increment (state) {
          // 这里的 `state` 对象是模块的局部状态
          state.count++
        }
      },
      getters: {
        doubleCount (state) {
          return state.count * 2
        }
      }
    }
    const moduleA = {
      namespaced: true,
      state: { count: 1, age1: 20 },
      getters: {
        doubleCount (state) {
          return state.count * 2
        }
      },
      modules: {
        c: moduleC
      }
    }
    const moduleB = {
      namespaced: true,
      state: { count: 1, age1: 20 },
      getters: {
        doubleCount (state) {
          return state.count * 2
        }
      }
    }
    const store = new Vuex.Store({
      state() {
        return {
            count: 0,
            todos: [
              { id: 1, text: '...', done: true },
              { id: 2, text: '...', done: false }
            ]
        }
      },
      getters: {
        doneTodos: state => {
          return state.todos.filter(todo => todo.done)
        }
      },
      modules: {
        a: moduleA,
        b: moduleB
      }
    })
    var vm = new Vue({
      el: '#example',
      data: {
        age: 10
      },
      store,
      mounted() {
        console.log(this.count)
        this.localeincrement('hehe')
        console.log(this.count)
      },
      // computed: Vuex.mapState('a', [
      //     'count', 'age1'
      //   ]
      // ),
      computed: Vuex.mapState([
          'count'
        ]
      ),
      methods: {
        ...Vuex.mapMutations({
          add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
        }),
        ...Vuex.mapMutations({
          localeincrement (commit, args) {
            console.log(commit)
            console.log(args)
            commit('increment', args)
          }
        })
      }
    })
    console.log(vm)

对应的根级别的getters:

![](/assets/vuex10.png)

**局部化state:**

state的局部化时在前述的local对象上定义state属性，并重新定义该属性的get函数：


	state: {
	    get: function () { return getNestedState(store.state, path); }
	}


它会调用getNestedState方法由根实例的state向下查找对应命名空间的state, getNestedState定义在VUEX源码的757 ~ 761 行：


	function getNestedState (state, path) {
	  return path.length
	    ? path.reduce(function (state, key) { return state[key]; }, state)
	    : state
	}


以上面的例子为例，我们来看一下a模块和c模块的state关系就知道了：

![](/assets/vuex11.png)

所有的局部化完成之后会将该local对象返回，挂载在根模块的context属性上：


	return local


我们来看看例子：

![](/assets/vuex12.png)

回过头来继续看installModule函数的执行，它会分别遍历mutaions, actions, getters，并分别执行registerMutation，registerAction，registerGetter：


	module.forEachMutation(function (mutation, key) {
	  var namespacedType = namespace + key;
	  registerMutation(store, namespacedType, mutation, local);
	});
	
	module.forEachAction(function (action, key) {
	  var type = action.root ? key : namespace + key;
	  var handler = action.handler || action;
	  registerAction(store, type, handler, local);
	});
	
	module.forEachGetter(function (getter, key) {
	  var namespacedType = namespace + key;
	  registerGetter(store, namespacedType, getter, local);
	});


我们来分别看一下：

**注册Mutation:**

注册mutaion首先会调用forEachMutation进行遍历：


	module.forEachMutation(function (mutation, key) {
	  var namespacedType = namespace + key;
	  registerMutation(store, namespacedType, mutation, local);
	});


forEachMutation的实现在VUEX源码的161 ~ 165行：


	Module.prototype.forEachMutation = function forEachMutation (fn) {
	  if (this._rawModule.mutations) {
	    forEachValue(this._rawModule.mutations, fn);
	  }
	};


forEachValue的实现我们在前面讲过，实际上就是遍历对象的key，将value，key作为参数应用于fn。而对于forEachAction而言，它的fn就是：


	function (action, key) {
	    var type = action.root ? key : namespace + key;
	    var handler = action.handler || action;
	    registerAction(store, type, handler, local);
	}


这里的核心还是归结到使用命名空间注册action了，它实际调用的是registerAction，该函数定义在VUEX源码的700 ~ 705 行：


	function registerMutation (store, type, handler, local) {
	  var entry = store._mutations[type] || (store._mutations[type] = []);
	  entry.push(function wrappedMutationHandler (payload) {
	    handler.call(store, local.state, payload);
	  });
	}


实际上，它是在根store的\_mutaions上以命名空间为key，注册对应的mutaion。稍后我们会有实际例子展示。

**注册Action:**

同注册Mutation原理一模一样，注册action首先会调用forEachAction进行遍历：


	module.forEachAction(function (action, key) {
	    var type = action.root ? key : namespace + key;
	    var handler = action.handler || action;
	    registerAction(store, type, handler, local);
	});


forEachAction的实现在VUEX源码的155 ~ 159行：

	Module.prototype.forEachAction = function forEachAction (fn) {
	  if (this._rawModule.actions) {
	    forEachValue(this._rawModule.actions, fn);
	  }
	};

forEachValue的实现我们在前面讲过，实际上就是遍历对象的key，将value，key作为参数应用于fn。而对于forEachMutation而言，它的fn就是：


	function (mutation, key) {
	  var namespacedType = namespace + key;
	  registerMutation(store, namespacedType, mutation, local);
	}


这里的核心还是归结到使用命名空间注册mutation了，它实际调用的是registerMutation，该函数定义在VUEX源码的707 ~ 730 行：


	function registerAction (store, type, handler, local) {
	  var entry = store._actions[type] || (store._actions[type] = []);
	  entry.push(function wrappedActionHandler (payload, cb) {
	    var res = handler.call(store, {
	      dispatch: local.dispatch,
	      commit: local.commit,
	      getters: local.getters,
	      state: local.state,
	      rootGetters: store.getters,
	      rootState: store.state
	    }, payload, cb);
	    if (!isPromise(res)) {
	      res = Promise.resolve(res);
	    }
	    if (store._devtoolHook) {
	      return res.catch(function (err) {
	        store._devtoolHook.emit('vuex:error', err);
	        throw err
	      })
	    } else {
	      return res
	    }
	  });
	}


实际上，它是在根store的\_actions上以命名空间为key，注册对应的action。但是action比mutaion复杂，主要体现在：

1. 在分发action时可以提供一个callback。
2. 在实际action的执行时，action的handle的参数会更复杂。

对于第二点，从代码中，我们可以看出，action实际上的会暴露四个参数：

1. 根级别的store。
2. 局部化的以及根级别的一些内容。这其中包括局部化的dispatch、commit、getters、state，根级别的getters、state。
3. 分发action时的载荷。
4. 分发action时的callback。

action和mutations的最大区别还在于，action是支持异步的。这在上述代码也有体现。

稍后我们会有实际例子展示注册action的效果。

**注册Getters:**

同注册Mutation、Action原理类似，注册getters首先会调用forEachGetter进行遍历：


	module.forEachGetter(function (getter, key) {
	    var namespacedType = namespace + key;
	    registerGetter(store, namespacedType, getter, local);
	});


forEachGetter的实现在VUEX源码的149 ~ 153行：


	Module.prototype.forEachGetter = function forEachGetter (fn) {
	  if (this._rawModule.getters) {
	    forEachValue(this._rawModule.getters, fn);
	  }
	};


forEachValue的实现我们在前面讲过，实际上就是遍历对象的key，将value，key作为参数应用于fn。而对于forEachGetter而言，它的fn就是：


	function (getter, key) {
	    var namespacedType = namespace + key;
	    registerGetter(store, namespacedType, getter, local);
	}


这里的核心还是归结到使用命名空间注册getters了，它实际调用的是registerGetter，该函数定义在VUEX源码的732 ~ 747 行：


	function registerGetter (store, type, rawGetter, local) {
	  if (store._wrappedGetters[type]) {
	    {
	      console.error(("[vuex] duplicate getter key: " + type));
	    }
	    return
	  }
	  store._wrappedGetters[type] = function wrappedGetter (store) {
	    return rawGetter(
	      local.state, // local state
	      local.getters, // local getters
	      store.state, // root state
	      store.getters // root getters
	    )
	  };
	}


实际上，它是在根store的\_wrappedGetters上以命名空间为key，注册对应的getters。它实际上是对getters对应的handle参数做了处理，暴露出四个参数：局部化的state、局部化的getters、根级别的state、根级别的getters。

以上Mutation、Action、Getters的注册可以通过下面这个例子来加深理解：

    const moduleC = {
      namespaced: true,
      state: { count: 1, age1: 20 },
      mutations: {
        increment (state) {
          // 这里的 `state` 对象是模块的局部状态
          state.count++
        }
      },
      getters: {
        doubleCount (state) {
          return state.count * 2
        }
      }
    }
    const moduleA = {
      namespaced: true,
      state: { count: 1, age1: 20 },
      mutations: {
        increment (state) {
          // 这里的 `state` 对象是模块的局部状态
          state.count++
        }
      },
      actions: {
        increment (context) {
          context.commit('increment')
        }
      },
      getters: {
        doubleCount (state) {
          return state.count * 2
        }
      },
      modules: {
        c: moduleC
      }
    }
    const moduleB = {
      namespaced: true,
      state: { count: 1, age1: 20 },
      mutations: {
        increment (state) {
          // 这里的 `state` 对象是模块的局部状态
          state.count++
        }
      },
      actions: {
        increment (context) {
          context.commit('increment')
        }
      },
      getters: {
        doubleCount (state) {
          return state.count * 2
        }
      }
    }

    const store = new Vuex.Store({
      state() {
        return {
            count: 0,
            todos: [
              { id: 1, text: '...', done: true },
              { id: 2, text: '...', done: false }
            ]
        }
      },
      mutations: {
        increment (state) {
          state.count++
        },
        increment1 (state) {
          state.count++
        }
      },
      getters: {
        doneTodos: state => {
          return state.todos.filter(todo => todo.done)
        }
      },
      actions: {
        increment (context) {
          context.commit('increment')
        }
      },
      modules: {
        a: moduleA,
        b: moduleB
      }
    })
    var vm = new Vue({
      el: '#example',
      data: {
        age: 10
      },
      store,
      mounted() {
        console.log(this.count)
        this.localeincrement('hehe')
        console.log(this.count)
      },
      // computed: Vuex.mapState('a', [
      //     'count', 'age1'
      //   ]
      // ),
      computed: Vuex.mapState([
          'count'
        ]
      ),
      methods: {
        ...Vuex.mapMutations({
          add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
        }),
        ...Vuex.mapMutations({
          localeincrement (commit, args) {
            console.log(commit)
            console.log(args)
            commit('increment', args)
          }
        })
      }
    })
    console.log(vm)

分别看看注册的结果：

![](/assets/vuex14.png)
#
![](/assets/vuex13.png)
#
![](/assets/vuex15.png)

回过头来继续看installModule的执行，它会遍历该模块的子模块，递归调用installModule来完成上述注册:

	module.forEachChild(function (child, key) {
	    installModule(store, rootState, path.concat(key), child, hot);
	});


而forEachChild定义在VUEX源码的145 ~ 147 行：

	Module.prototype.forEachChild = function forEachChild (fn) {
	  forEachValue(this._children, fn);
	};


至此，installModule的执行就jies 了，我们回过头继续看看Store类的构造函数执行，接下来执行的是resetStoreVM函数，它定义在VUEX源码的531 ~ 575 行：


	function resetStoreVM (store, state, hot) {
	  var oldVm = store._vm;
	
	  // bind store public getters
	  store.getters = {};
	  var wrappedGetters = store._wrappedGetters;
	  var computed = {};
	  forEachValue(wrappedGetters, function (fn, key) {
	    // use computed to leverage its lazy-caching mechanism
	    computed[key] = function () { return fn(store); };
	    Object.defineProperty(store.getters, key, {
	      get: function () { return store._vm[key]; },
	      enumerable: true // for local getters
	    });
	  });
	
	  // use a Vue instance to store the state tree
	  // suppress warnings just in case the user has added
	  // some funky global mixins
	  var silent = Vue.config.silent;
	  Vue.config.silent = true;
	  store._vm = new Vue({
	    data: {
	      $$state: state
	    },
	    computed: computed
	  });
	  Vue.config.silent = silent;
	
	  // enable strict mode for new vm
	  if (store.strict) {
	    enableStrictMode(store);
	  }
	
	  if (oldVm) {
	    if (hot) {
	      // dispatch changes in all subscribed watchers
	      // to force getter re-evaluation for hot reloading.
	      store._withCommit(function () {
	        oldVm._data.$$state = null;
	      });
	    }
	    Vue.nextTick(function () { return oldVm.$destroy(); });
	  }
	}


它实际上主要做的是在store的实例上定义vm属性，而vm上挂载的是一个新的Vue实例，这个vue实例的data为store的state，而computed为store的getters，我们同样以前面的那个例子，看看效果：

![](/assets/vuex16.png)

Store构造函数的最后是和插件、调试工具有关的几行代码，这里不再细述：

	// apply plugins
	plugins.forEach(function (plugin) { return plugin(this$1); });
	
	if (Vue.config.devtools) {
	  devtoolPlugin(this);
	}

## 4.2 原型函数

### 4.2.1 Store.prototype.commit
commit定义在VUEX源码的376 ~ 409行：

	Store.prototype.commit = function commit (_type, _payload, _options) {
	    var this$1 = this;
	
	  // check object-style commit
	  var ref = unifyObjectStyle(_type, _payload, _options);
	    var type = ref.type;
	    var payload = ref.payload;
	    var options = ref.options;
	
	  var mutation = { type: type, payload: payload };
	  var entry = this._mutations[type];
	  if (!entry) {
	    {
	      console.error(("[vuex] unknown mutation type: " + type));
	    }
	    return
	  }
	  this._withCommit(function () {
	    entry.forEach(function commitIterator (handler) {
	      handler(payload);
	    });
	  });
	  this._subscribers.forEach(function (sub) { return sub(mutation, this$1.state); });
	
	  if (
	    "development" !== 'production' &&
	    options && options.silent
	  ) {
	    console.warn(
	      "[vuex] mutation type: " + type + ". Silent option has been removed. " +
	      'Use the filter functionality in the vue-devtools'
	    );
	  }
	};

它所做的就是当mutation被提交时执行对应的函数，并且还会执行订阅列表里面的回调函数。

### 4.2.2 Store.prototype.dispatch
dispatch定义在VUEX源码的411 ~ 433 行，它的原理和commit基本上一样的,也是在分发action时执行对应的函数，并且执行订阅action的列表，所不同的是action是支持异步的：

	Store.prototype.dispatch = function dispatch (_type, _payload) {
	    var this$1 = this;
	
	  // check object-style dispatch
	  var ref = unifyObjectStyle(_type, _payload);
	    var type = ref.type;
	    var payload = ref.payload;
	
	  var action = { type: type, payload: payload };
	  var entry = this._actions[type];
	  if (!entry) {
	    {
	      console.error(("[vuex] unknown action type: " + type));
	    }
	    return
	  }
	
	  this._actionSubscribers.forEach(function (sub) { return sub(action, this$1.state); });
	
	  return entry.length > 1
	    ? Promise.all(entry.map(function (handler) { return handler(payload); }))
	    : entry[0](payload)
	};

### 4.2.3 Store.prototype.subscribe
subscribe定义在VUEX源码的435 ~ 437行，用于注册订阅mutation的回调：

	Store.prototype.subscribe = function subscribe (fn) {
	  return genericSubscribe(fn, this._subscribers)
	};

官方描述如下：

>注册监听 store 的 mutation。handler 会在每个 mutation 完成后调用，接收 mutation 和经过 mutation 后的状态作为参数：
>
	store.subscribe((mutation, state) => {
	  console.log(mutation.type)
	  console.log(mutation.payload)
	})
通常用于插件。

它实际上调用的是定义在VUEX源码507 ~ 517行的genericSubscribe函数：

	function genericSubscribe (fn, subs) {
	  if (subs.indexOf(fn) < 0) {
	    subs.push(fn);
	  }
	  return function () {
	    var i = subs.indexOf(fn);
	    if (i > -1) {
	      subs.splice(i, 1);
	    }
	  }
	}

它实际上就是订阅mutation，并将回调放入_subscribers订阅列表中，它会返回一个函数，用于解除订阅。这个主要用在调试工具里。

### 4.2.4 Store.prototype.subscribeAction
subscribeAction定义在VUEX源码的439 ~ 441行，用于注册订阅action的回调，它和subscribe函数的原理是一模一样的：

	Store.prototype.subscribeAction = function subscribeAction (fn) {
	  return genericSubscribe(fn, this._actionSubscribers)
	};

它实际上也调用的是定义在VUEX源码507 ~ 517行的genericSubscribe函数,这个在前面已经讲过了。它实际上就是订阅action，并将回调放入_actionSubscribers订阅列表中，它会返回一个函数，用于解除订阅。这个也主要用在调试工具里。

### 4.2.5 Store.prototype.watch
watch定义在VUEX源码的433 ~ 450行：

	Store.prototype.watch = function watch (getter, cb, options) {
	    var this$1 = this;
	
	  {
	    assert(typeof getter === 'function', "store.watch only accepts a function.");
	  }
	  return this._watcherVM.$watch(function () { return getter(this$1.state, this$1.getters); }, cb, options)
	};

我们来直接看看官方文档对它的解释吧：

>响应式地监测一个 getter 方法的返回值，当值改变时调用回调函数。getter 接收 store 的状态作为唯一参数。接收一个可选的对象参数表示 Vue 的 vm.$watch 方法的参数。
要停止监测，直接调用返回的处理函数。

### 4.2.6 Store.prototype.replaceState
replcaeState定义在VUEX源码的452 ~ 489行，用于替换_vm属性上存储的状态：

	Store.prototype.replaceState = function replaceState (state) {
	    var this$1 = this;
	
	  this._withCommit(function () {
	    this$1._vm._data.$$state = state;
	  });
	};

### 4.2.7 Store.prototype.registerModule
registerModule定义在VUEX源码的第460 ~ 474 行，使得Store实例能够在给定路径注册相应的模块，实际上还是从根模块开始，找到对应的路径，然后注册。注册完成后需要重新安装模块，然后重置_vm属性：

	Store.prototype.registerModule = function registerModule (path, rawModule, options) {
	    if ( options === void 0 ) options = {};
	
	  if (typeof path === 'string') { path = [path]; }
	
	  {
	    assert(Array.isArray(path), "module path must be a string or an Array.");
	    assert(path.length > 0, 'cannot register the root module by using registerModule.');
	  }
	
	  this._modules.register(path, rawModule);
	  installModule(this, this.state, path, this._modules.get(path), options.preserveState);
	  // reset store to update getters...
	  resetStoreVM(this, this.state);
	};

### 4.2.8 Store.prototype.unregisterModule
unregisterModule定义在VUEX源码的第476 ~ 491行，它使得Store实例可以通过提供的path参数解除对应模块的注册。实际上它还是根据path找到对应的模块的父模块，然后调用父模块的unregister方法完成解绑：

	Store.prototype.unregisterModule = function unregisterModule (path) {
	    var this$1 = this;
	
	  if (typeof path === 'string') { path = [path]; }
	
	  {
	    assert(Array.isArray(path), "module path must be a string or an Array.");
	  }
	
	  this._modules.unregister(path);
	  this._withCommit(function () {
	    var parentState = getNestedState(this$1.state, path.slice(0, -1));
	    Vue.delete(parentState, path[path.length - 1]);
	  });
	  resetStore(this);
	};


### 4.2.9 Store.prototype.hotUpdate
hotUpdate定义在VUEX源码的493 ~ 496行：

	Store.prototype.hotUpdate = function hotUpdate (newOptions) {
	  this._modules.update(newOptions);
	  resetStore(this, true);
	};

hotUpdate可以热更新整个模块，跟新完后调用resetStore重置整个模块，resetStore的定义在519 ~ 529行：

	function resetStore (store, hot) {
	  store._actions = Object.create(null);
	  store._mutations = Object.create(null);
	  store._wrappedGetters = Object.create(null);
	  store._modulesNamespaceMap = Object.create(null);
	  var state = store.state;
	  // init all modules
	  installModule(store, state, [], store._modules.root, true);
	  // reset vm
	  resetStoreVM(store, state, hot);
	}
可以看到基本上就是将构造函数推倒重来了一遍。

### 4.2.10 Store.prototype.\_withCommit

\_withCommit定义在VUEX源码的498 ~ 503行：

	Store.prototype._withCommit = function _withCommit (fn) {
	  var committing = this._committing;
	  this._committing = true;
	  fn();
	  this._committing = committing;
	};

它首先设置当前store的committing状态为true，表示正在commit，然后执行对应的函数，当执行完毕后，重置commit状态。

