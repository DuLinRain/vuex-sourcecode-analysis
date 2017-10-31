# 第二章 Module类

Module类定义在VUEX源码的106 ~ 168 行，我们来具体看看它的内容。

## 2.1 成员属性

Module类的成员属性有四个，分别是：

1. runtime。表示是否运行时，类型Boolean。
2. \_children。存储该模块的直接子模块，类型Object。
3. \_rawModule。存储该模块自身，类型Object。
4. state。存储该模块的state，类型Object。

其源码定义在106 ~ 112 行：

```
var Module = function Module (rawModule, runtime) {
  this.runtime = runtime;
  this._children = Object.create(null);
  this._rawModule = rawModule;
  var rawState = rawModule.state;
  this.state = (typeof rawState === 'function' ? rawState() : rawState) || {};
};
```

该类的定义无特殊之处，但从中我们可以看出，定义模块的`state`时，**并不一定需要它是一个对象，它也可以是返回一个对象的工厂函数**。因为从代码的执行来看，当`state`属性的值是一个函数时，会把这个函数的执行结果作为`state`。这一点在官方文档中是没有提及的。

我们可以通过下面这个例子来证明：

```
const store = new Vuex.Store({
  state() {
    return {
        count: 0,
        todos: [
          { id: 1, text: '...', done: true },
          { id: 2, text: '...', done: false }
        ]
    }
  }
})
console.log(store)
```

其输出结果如下：![](/assets/vuex1.png)

## 2.2 原型函数

Module原型上分别定义了：

1. 操作\_children属性的addChild、removeChild、getChild、forEachChild四个方法。
2. 操作\_rawModule属性的update、forEachGetter、forEachAction、forEachMutation四个方法。

我们来分别看一看这几个方法的实现。

#### 2.2.1 Module.prototype.addChild方法

addChild方法定义在VUEX源码的120 ~ 122 行，它的实现比较简单，就是将模块名称作为key，模块内容作为value定义在父模块的\_children对象上：

```
Module.prototype.addChild = function addChild (key, module) {
  this._children[key] = module;
};
```

#### 2.2.2 Module.prototype.removeChild方法

removeChild方法定义在VUEX源码的124 ~ 126 行，它的实现也比较简单，就是采用删除对象属性的方法，将定义在父模块\_children属性上的子模块delete：

```
Module.prototype.removeChild = function removeChild (key) {
  delete this._children[key];
};
```

#### 2.2.3 Module.prototype.getChild方法

getChild方法定义在VUEX源码的128 ~ 130 行，它的实现也比较简单，就是将父模块\_children属性的子模块查找出来并return出去：

```
Module.prototype.getChild = function getChild (key) {
  return this._children[key]
};
```

#### 2.2.4 Module.prototype.forEachChild方法

forEachChild方法定义在VUEX源码的145 ~ 147 行，它接受一个函数作为参数，并且将该函数应用在Module实例的\_children属性上，也就是说会应用在所有的子模块上：

```
Module.prototype.forEachChild = function forEachChild (fn) {
  forEachValue(this._children, fn);
};
```

可以看到forEachChild 方法实际上是调用了另外一个辅助函数forEachValue，这个函数接收Module实例的\_children属性以及forEachChild 方法的fn参数作为参数。它实际上是遍历\_children对象，并将value和key作为fn的参执行fn。它定义在VUEX源码的87 ~ 92 行，我们来看看它的实现：

```
/**
 * forEach for object
 */
function forEachValue (obj, fn) {
  Object.keys(obj).forEach(function (key) { return fn(obj[key], key); });
}
```

#### 2.2.5 Module.prototype.update方法

update方法定义在VUEX源码的132 ~ 143 行，它接收一个模块，然后用该模块更新当前Module实例上的\_rawModule属性。更新的内容包括\_rawModule的namespaced、actions、mutations、getters：

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

#### 2.2.6 Module.prototype.forEachGetter方法

forEachGetter方法定义在VUEX源码的149 ~ 153 行，同forEachChild方法原理类似，forEachGetter方法接受一个函数作为参数，并且将该函数应用在Module实例的\_rawModule属性的getters上，也就是说会应用在该模块的所有getters上：

```
Module.prototype.forEachGetter = function forEachGetter (fn) {
  if (this._rawModule.getters) {
    forEachValue(this._rawModule.getters, fn);
  }
};
```

可以看到forEachGetter 方法实际上是也调用了另外一个辅助函数forEachValue，这个forEachValue函数前面已经介绍过，这里就不再赘述。

#### 2.2.7 Module.prototype.forEachAction方法

forEachAction方法定义在VUEX源码的155 ~ 159 行，同forEachChild、forEachGetter 方法原理类似，forEachAction方法接受一个函数作为参数，并且将该函数应用在Module实例的\_rawModule属性的actions上，也就是说会应用在该模块的所有actions上：

```
Module.prototype.forEachAction = function forEachAction (fn) {
  if (this._rawModule.actions) {
    forEachValue(this._rawModule.actions, fn);
  }
};
```

#### 2.2.8 Module.prototype.forEachMutation方法

forEachMutation方法定义在VUEX源码的161 ~ 165 行，同forEachChild、forEachGetter、 forEachGetter 方法原理类似，forEachMutation方法接受一个函数作为参数，并且将该函数应用在Module实例的\_rawModule属性的mutations上，也就是说会应用在该模块的所有mutations上：

```
Module.prototype.forEachMutation = function forEachMutation (fn) {
  if (this._rawModule.mutations) {
    forEachValue(this._rawModule.mutations, fn);
  }
};
```

由于forEachChild、forEachGetter、 forEachGetter、forEachMutation方法类似，所以我们这里仅以forEachMutation方法举一个例子，说明它及其实际执行者forEachValue的工作原理：

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
        }
    }
    var moduleIns = new Module(options)
    moduleIns.forEachMutation(function (value, key) {
        console.log(`mutations key is : ${key}`)
        console.log(`mutations value is : ${value}`)
    })

这里我们只是为了描述其原理，所以上述例子仅仅只是输出了motations的key和value，实际的使用场合会比这复杂的多，我们来看看上述例子的输出结果：

```
mutations key is : increment
mutations value is : increment(state) {
  state.count++
}
mutations key is : increment1
mutations value is : increment1(state) {
  state.count++
}
```



