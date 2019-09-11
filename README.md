 

# reselect explain

```javascript
只有100来行代码的库----https://github.com/reduxjs/reselect#connecting-a-selector-to-the-redux-store
```

1.首先自己了解它，至少你要知道它的用法吧！

举个列子：

```javascript
import { createSelector } from 'reselect'
```

核心函数：使用之前调用这个函数 createSelector

在redux中，mapStateToProps （一般都是我们作为常用的数据返回函数，具体用法不过多解释）
```javscript
const mapStateToProps = (state, props) => {
        const { initData} = state;
        return { initData };
      };
```
initData 作为一个数据返回体 ， 正常情况下我们对其进行操作处理返回会这样做
```javscript
const mapStateToProps = (state, props) => {
        const {
             initData
        } = state;
        return {
              initData：initData.filter((item)=>item%2)
        };
    };
```
其中返回函数中进行了js操作（filter) ，对应Reselect-github说法

```javscript
Connecting a Selector to the Redux Store

const makeMapStateToProps = () => {
  const getVisibleTodos = makeGetVisibleTodos()
  const mapStateToProps = (state, props) => {
    return {
      todos: getVisibleTodos(state, props)  //在此进行操作是一个非常棒的做法（他自己说的）
    }
  }
  return mapStateToProps
}
```

这样就会产生一个问题，每次调用都是进行相应的操作计算返回（这个也有介绍），数据量达到一个量级的时候就会严重影响性能。
Reselect 的作用：

createSelector（） ：
内部参数接受对应源码来看
```javscript
export const createSelector = createSelectorCreator(defaultMemoize)
```
reselect 导出了这个函数  createSelector 并且这个函数实质返回的是一个createSelectorCreator函数

对应源码52行：
createSelectorCreator（）：
该函数内部返回一个函数作为我们使用的时候参入的参数（...funcs-可扩展的参数集合返回的是一个数组)
现在看下作者内部规定我们该怎么传入参数：
看github介绍 
```javascript
selectors/todoSelectors.js
import { createSelector } from 'reselect'

const getVisibilityFilter = (state, props) =>
  state.todoLists[props.listId].visibilityFilter

const getTodos = (state, props) =>
  state.todoLists[props.listId].todos

const makeGetVisibleTodos = () => {
  return createSelector(
    [ getVisibilityFilter, getTodos ],
    (visibilityFilter, todos) => {
      switch (visibilityFilter) {
        case 'SHOW_COMPLETED':
          return todos.filter(todo => todo.completed)
        case 'SHOW_ACTIVE':
          return todos.filter(todo => !todo.completed)
        default:
          return todos
      }
    }
  )
}

export default makeGetVisibleTodos
```
函数接收的是两个参数，第一个是我们要缓存的数据值，第二个是我们数据操作的函数 <br/>
#### ok  look down ||
```javascript
 const resultFunc = funcs.pop();
 const dependencies = getDependencies(funcs);
```
源码中
funcs.pop() 等于resultFunc  ，实质是获取到了我们第二个参数（数据操作函数）
第二行实质是常量 dependencies  = 我们传入的第一个参数（需要缓存的数据），并且看源码调用了一个工具函数getDependencies
   ```javascript
function getDependencies(funcs) {
  const dependencies = Array.isArray(funcs[0]) ? funcs[0] : funcs

  if (!dependencies.every(dep => typeof dep === 'function')) {
    const dependencyTypes = dependencies.map(
      dep => typeof dep
    ).join(', ')
    throw new Error(
      'Selector creators expect all input-selectors to be functions, ' +
      `instead received the following types: [${dependencyTypes}]`
    )
  }

  return dependencies
}
```
该函数处理了我们第一个参数（数组集合） 
```javascript
  Array.isArray(funcs[0]) ? funcs[0] : funcs
```
如果我们参数第一个是数组则返回数据第一个元素，否则全部返回我们的第一个参数（数组）
```javascript
 dependencies.every(dep => typeof dep === 'function')
```
判断每一个元素是否都是函数，如果不是就会报错
----------简而言之，就是一个判断是否是函数的一个工具函数 <br/>
### ok  look down 
闭包来了  
```javascript
const memoizedResultFunc = memoize(
      function () {
        recomputations++
        return resultFunc.apply(null, arguments)
      },
      ...memoizeOptions
    )
```

此时：memoizedResultFunc 得到了一个 闭包函数等待执行

```javascript
  const selector = memoize(function () {
      const params = []
      const length = dependencies.length
      for (let i = 0; i < length; i++) {
        params.push(dependencies[i].apply(null, arguments))
      }
      return memoizedResultFunc.apply(null, params)
    })
```
params 获取的就是我们传入的第一个参数集合 ==【】
 memoizedResultFunc.apply(null, params) ：memoizedResultFunc函数通过apply 获取到了params参数
 
此时执行进入 memoize ----- defaultMemoize
memoize调用的就是这个函数，第一个参数就是一个函数，
memoizedResultFunc.apply(null, 获取到了params参数)执行结果
是不是很熟悉这个结果执行函数，它就是我们的数据操作函数（第二个参数）。
给到了defaultMemoize函数的第一个参数func
现在记住了，我们要操作的结果（也是我们自己需要得到的结果函数）在defaultMemoize函数中
通过参数func
指给了 lastResult = func.apply(null, arguments) 结果（） 它；进行第一步缓存数据lastResult(闭包生效了）

memoize 是我们核心函数的默认参数 defaultMemoize
重要的部分就是这里了
```javascript
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null
  let lastResult = null
  return function () {
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      lastResult = func.apply(null, arguments)
    }
    lastArgs = arguments   
    return lastResult
  }
}
```
###重点：arguments是执行函数绑定了params的值

需要注意的一点 ，函数areArgumentsShallowlyEqual 是一个比对过程
看源码
```javascript
function areArgumentsShallowlyEqual(equalityCheck, prev, next) {
  if (prev ==prev= null || next === null || prev.length !== next.length) {
    return false
  }
  const length = prev.length
  for (let i = 0; i < length; i++) {
    if (!equalityCheck(prev[i], next[i])) {
      return false
    }
  }
  return true
}
```
这个意思是：三个参数接收（第一个是比对方法，默认是===，可自己配置，第二个参数prev---是我们的
lastResult结果集合 (  lastArgs = arguments)    ）
如果 (prev ==prev= null || next === null ）为空 并且 prev.length !== next.length 
说明是初始或者发生改变 返回false 
源码中
```haml
 if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      lastResult = func.apply(null, arguments)
    }
```
进行缓存赋值 操作 ！false ===true 拿到的值是我们操作函数的值 ,否则返回（页面的原始数据）

如果三个条件都不满足说明：反的来说就是需要改变了
遍历prev (已经是页面中的数据或则是操作函数返回的数据) 比对 next（arguments,我们第一个参数
的数据也是页面中的原始数据）
```javascript
 for (let i = 0; i < length; i++) {
    if (!equalityCheck(prev[i], next[i])) {
      return false
    }
  }
```
这个就是比对函数的作用  缓存的判断处理 <br/>
  // apply arguments instead of spreading for performance.


最后就是页面中的 return selector;














 
