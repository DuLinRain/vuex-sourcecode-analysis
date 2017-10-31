# 第三章 ModuleCollection类

ModuleCollection类定义在VUEX源码的169 ~ 251 行，我们来具体看看它的内容。

## 3.1 成员属性

Module类的成员属性只有1个：

1. root。挂载着根模块。

它并不是直接在构造函数中显示定义的，而是在原型函数`register`中定义的。通过在构造函数中调用`register`函数从而在成员属性`root`上挂载根模块。其实现在VUEX源码的192 ~ 214 行：

```
ModuleCollection.prototype.register = function register (path, rawModule, runtime) {
    var this$1 = this;
    if ( runtime === void 0 ) runtime = true;

  {
    assertRawModule(path, rawModule);
  }

  var newModule = new Module(rawModule, runtime);
  if (path.length === 0) {
    this.root = newModule;
  } else {
    var parent = this.get(path.slice(0, -1));
    parent.addChild(path[path.length - 1], newModule);
  }

  // register nested modules
  if (rawModule.modules) {
    forEachValue(rawModule.modules, function (rawChildModule, key) {
      this$1.register(path.concat(key), rawChildModule, runtime);
    });
  }
};
```

## 3.2 原型函数

### 3.2.1 ModuleCollection.prototype.get方法

get方法定义在VUEX源码的174 ~ 178 行，get方法主要是根据给定的模块名（模块路径），从根store开始，逐级向下查找对应的模块找到最终的那个模块，它的核心是采用的reduce函数来实现的，我们来看看它的源码：

```
ModuleCollection.prototype.get = function get (path) {
  return path.reduce(function (module, key) {
    return module.getChild(key)
  }, this.root)
};
```

### 3.2.2 ModuleCollection.prototype.getNamespace方法

getNamespace方法定义在VUEX源码的180 ~ 186 行，getNamespace方法同样是根据给定的模块名（模块路径），从根store开始，逐级向下生成该模块的命名空间，当途中所遇到的模块没有设置namespaced属性的时候，其命名空间默认为空字符串，而如果设置了namespaced属性，则其命名空间是模块名+反斜线\(/\)拼接起来的字符。我们来看看它的源码实现：

```
ModuleCollection.prototype.getNamespace = function getNamespace (path) {
  var module = this.root;
  return path.reduce(function (namespace, key) {
    module = module.getChild(key);
    return namespace + (module.namespaced ? key + '/' : '')
  }, '')
};
```

### 3.2.3 ModuleCollection.prototype.update方法

update方法定义在VUEX源码的188 ~ 190 行，用于从根级别开始逐级更新模块的内容：

```
ModuleCollection.prototype.update = function update$1 (rawRootModule) {
  update([], this.root, rawRootModule);
};
```

它实际调用的是定义在VUEX源码224 ~ 251 行的全局update方法：

```
function update (path, targetModule, newModule) {
  {
    assertRawModule(path, newModule);
  }

  // update target module
  targetModule.update(newModule);

  // update nested modules
  if (newModule.modules) {
    for (var key in newModule.modules) {
      if (!targetModule.getChild(key)) {
        {
          console.warn(
            "[vuex] trying to add a new module '" + key + "' on hot reloading, " +
            'manual reload is needed'
          );
        }
        return
      }
      update(
        path.concat(key),
        targetModule.getChild(key),
        newModule.modules[key]
      );
    }
  }
}
```

这个方法会调用指定模块（第二个参数）的update方法，我们在第二章介绍过，每个Module实例都有一个update原型方法，定义在132 ~ 143行，这里再一次粘贴如下：

```
Module.prototype.update = function update (rawModule) {
  this._rawModule.namespaced = rawModule.namespaced;
  if (rawModule.actions) {
    this._rawModule.actions = rawModule.actions;
  }
  if (rawModule.mutations) {
    this._rawModule.mutations = rawModule.mutations;
  }
  if (rawModule.getters) {
    this._rawModule.getters = rawModule.getters;
  }
};
```

回到全局update方法，当判断出它还有子模块的时候，则会递归地调用update方法进行模块更新对应子模块。

### 3.2.4 ModuleCollection.prototype.register方法

register方法定义在VUEX源码的192 ~ 214 行，它主要是从根级别开始，逐级注册子模块，最终的模块链条会挂载在ModuleCollection实例的成员属性root上，我们来看看它的源码：

```
ModuleCollection.prototype.register = function register (path, rawModule, runtime) {
    var this$1 = this;
    if ( runtime === void 0 ) runtime = true;

  {
    assertRawModule(path, rawModule);
  }

  var newModule = new Module(rawModule, runtime);
  if (path.length === 0) {
    this.root = newModule;
  } else {
    var parent = this.get(path.slice(0, -1));
    parent.addChild(path[path.length - 1], newModule);
  }

  // register nested modules
  if (rawModule.modules) {
    forEachValue(rawModule.modules, function (rawChildModule, key) {
      this$1.register(path.concat(key), rawChildModule, runtime);
    });
  }
};
```

我们来过一遍该代码，这段代码首先保存了this副本：

```
var this$1 = this;
```

然后会判断runtime参数是否传递，如果没有传递，则会给它默认设置为ture：

```
if ( runtime === void 0 ) runtime = true;
{
  assertRawModule(path, rawModule);
}
```

这里有一个小知识点是采用void 0 判断undfined，这是一种很好的方法，具体原因可以参考本人的这篇文章“JavaScrip中如何正确并优雅地判断undefined”。接下来会实例化一个Module，对于根Module而言，它会被挂载到ModuleCollection实例的root成员属性上，而对于子模块，它会找到它的父模块，然后挂载到父模块的`_children`属性上，由Module类我们知道，每一个模块都会有一个`_children`属性，用于存储它的子模块：

```
var newModule = new Module(rawModule, runtime);
if (path.length === 0) {
  this.root = newModule;
} else {
  var parent = this.get(path.slice(0, -1));
  parent.addChild(path[path.length - 1], newModule);
}
```

最后面的代码就是递归的调用register函数用来执行前面几个步骤：

```
// register nested modules
if (rawModule.modules) {
  forEachValue(rawModule.modules, function (rawChildModule, key) {
    this$1.register(path.concat(key), rawChildModule, runtime);
  });
}
```

那么register函数是在哪里调用的呢？它是在VUEX源码169 ~ 172行ModuleCollection的构造函数中调用的，我们来看看：

```
var ModuleCollection = function ModuleCollection (rawRootModule) {
  // register root module (Vuex.Store options)
  this.register([], rawRootModule, false);
};
```

我们可以通过一个例子来直观地看一看：

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
      getters: {
        doubleCount (state) {
          return state.count * 2
        }
      }
    }
    var options = {
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
    }
    var moduleCollectionIns = new ModuleCollection(options)
    console.log(moduleCollectionIns)

输出结果：

![](/assets/vuex7.png)

### 3.2.5 ModuleCollection.prototype.unregister方法

unrigister方法定义在VUEX源码的216 ~ 222 行，它用于取消注册某个模块：

```
ModuleCollection.prototype.unregister = function unregister (path) {
  var parent = this.get(path.slice(0, -1));
  var key = path[path.length - 1];
  if (!parent.getChild(key).runtime) { return }

  parent.removeChild(key);
};
```

当取消注册某个模块时需要先拿到该模块的父模块，然后在父模块的\_children对象中删除该模块，调用的是父模块的removeChild方法。这里面会判断待取消模块是否处于运行时（runtime），当不处于运行时（runtime）时可以取消注册。



