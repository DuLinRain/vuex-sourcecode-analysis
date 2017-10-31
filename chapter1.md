# 第一章 概述

## 1.1 Vuex是什么？

按照官方的说法：

> Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。

## 1.2 Vuex导出了什么？

通常我们在实例化一个`store`的时候都是采用的下面这种方式：

```
import Vuex from 'vuex'

const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})
```

有时候我们将VUEX的`state`混入到计算属性时会采用这种方式：

    // 在单独构建的版本中辅助函数为 Vuex.mapState
    import { mapState } from 'vuex'

    export default {
      // ...
      computed: mapState({
        // 箭头函数可使代码更简练
        count: state => state.count,

        // 传字符串参数 'count' 等同于 `state => state.count`
        countAlias: 'count',

        // 为了能够使用 `this` 获取局部状态，必须使用常规函数
        countPlusLocalState (state) {
          return state.count + this.localCount
        }
      })
    }

很明显，由上面代码可以看出，VUEX导出了一个对象，而`Store、mapState`都只是这个对象的属性而已。那么，我们不禁会产生疑问：VUEX到底导出了些什么东西呢？

我们看看VUEX的源代码的 925 ~ 937 行：

```
var index = {
  Store: Store,//Store类
  install: install,//install方法
  version: '2.4.1',
  mapState: mapState,
  mapMutations: mapMutations,
  mapGetters: mapGetters,
  mapActions: mapActions,
  createNamespacedHelpers: createNamespacedHelpers//基于命名空间的组件绑定辅助函数
};

return index;
```

VUEX采用的是典型的IIFE\(立即执行函数表达式\)模式，当代码被加载\(通过`<script>`或`Vue.use()`\)后，VUEX会返回一个对象，这个对象包含了`Store`类、`install`方法、`mapState`辅助函数、`mapMutations`辅助函数、`mapGetters`辅助函数、`mapActions`辅助函数、`createNamespacedHelpers`辅助函数以及当前的版本号`version`。

## 1.2 Vuex源码的整体结构是怎样的？

纵观整个代码解构可以得出如下结论：

1. 代码12 ~ 105 行定义了一些类型检测、遍历之类的辅助函数。
2. 代码106 ~ 168 行定义了`Module`类。
3. 代码169 ~ 251 行定义了`ModuleCollection`类。
4. 代码253 ~ 293 行定义了几个断言辅助函数。
5. 代码294 ~ 775 行定义了`Store`类。
6. 代码777 ~ 788 行定义了`install`方法。
7. 代码790 ~ 815 行定义了`mapState`辅助函数。
8. 代码817 ~ 841 行定义了`mapMutations`辅助函数。
9. 代码843 ~ 864 行定义了`mapGetters`辅助函数。
10. 代码866 ~ 890 行定义了`mapActions`辅助函数。
11. 代码892 ~ 924 行定义了生成`mapState`、`mapMutations`、`mapGetters`、`mapActions`的辅助函数。
12. 代码925 ~ 937 行对主要内容进行了导出。

从以上分析可以看出，VUEX源码主要由`Store`类和`mapState`、`mapMutations`、`mapGetters`、`mapActions`四个辅助函数组成，其中`Store`类又由`ModuleCollection`实例组成，`ModuleCollection`类又由`Module`实例组成。VUEX源码就是通过这样的关系组织起来的。

