## 介绍

目前为止的操作都是 dispatch 单个 action 并立即响应，但实际程序应用场景会更复杂，例如 http 请求前发出一个 action 表示准备加载数据，收到响应后发出一个 action 表示请求成功或失败。

你可以直接 dispatch action，如 jQuery ajax 的 `beforeSend` 和 `complete` 里 dispatch action。但更建议使用 redux 提供的方式，使用 `redux-thunk` middleware。通过这个 middleware 可以将多个 dispatch 合并到一个 action creator 中，后面会说明什么是 middleware。


## Action

首先来分析需要怎么的 action。

对于异步操作，我们用三种 action 表示他，分别是表示发出请求，请求成功，请求失败。在不同的时间点分发 action。

```javascript
{ type: 'FETCH_POSTS' }
{ type: 'FETCH_POSTS', status: 'error', error: 'Oops' }
{ type: 'FETCH_POSTS', status: 'success', response: { ... } }
```

或者


```javascript
{ type: 'FETCH_POSTS_REQUEST' }
{ type: 'FETCH_POSTS_FAILURE', error: 'Oops' }
{ type: 'FETCH_POSTS_SUCCESS', response: { ... } }
```

对于使用一种类型的以状态区分作用还是分为三个类型都行，这里我选择使用分为三个类型。

### 同步 action creator

现在我们来定义程序要用到的 action 和 action creator：

`action.js`

```javascript
// 选中指定栏目
export const SELECT_SUBREDDIT = 'SELECT_SUBREDDIT'

export function selectSubreddit(subreddit) {
  return {
    type: SELECT_SUBREDDIT,
    subreddit
  }
}
```

按下刷新按钮更新内容

```javascript
export const INVALIDATE_SUBREDDIT = 'INVALIDATE_SUBREDDIT'

export function invalidateSubreddit(subreddit) {
  return {
    type: INVALIDATE_SUBREDDIT,
    subreddit
  }
}
```

上面是用户交互产生的 action，我们还有其它类型的 action，由网络请求产生，稍后将会看到如何 dispatch 他们，但现在只是定义。

当需要获取一些栏目，我们会 dispatch `REQUEST_POSTS` action:

```javascript
export const REQUEST_POSTS = 'REQUEST_POSTS'

export function requestPosts(subreddit) {
  return {
    type: REQUEST_POSTS,
    subreddit
  }
}
```

将 `REQUEST_POSTS` 从  `SELECT_SUBREDDIT` 或 `INVALIDATE_SUBREDDIT` 分离是非常重要的。他可能会接二连三的触发。当应用程序更复杂时，你可能想根据用户操作获取一些数据 (例如，提前刷新最流行的子栏目，或间隔一段时间刷新过时数据) 。你可能也想根据路由的变化获取数据，因此过早的捕获 UI 事件是不明智的。

最后，当网络请求响应，我们会 dispatch `RECEIVE_POSTS` :

```javascript
export const RECEIVE_POSTS = 'RECEIVE_POSTS'

export function receivePosts(subreddit, json) {
  return {
    type: RECEIVE_POSTS,
    subreddit,
    posts: json.data.children.map(child => child.data),
    receivedAt: Date.now()
  }
}
```

现在我们需要知道的只有这些，具体的 dispatch 机制与网络请求的关系我们稍后讨论。

> **错误处理**
> 在真实应用程序，我们想要处理请求出现错误的情况，在这里我们不会处理，在 [real world example](http://redux.js.org/docs/introduction/Examples.html#real-world) 展示了可能出现的一种情款。

## 设计 State 原形

像之前的文档一样，在实现代码之前要设计 State 的原型，异步代码更需要关心这一点，因此我们要好好想清楚。

我们从最通用的案例开始：列表。web 应用程序通常使用列表列出清单。例如文章列表，好友列表，你需要配置列表可以显示的内容。你想把他们分割到不同的 state，一遍单独更新获取新数据。

这是 `Reddit headlines` 程序的 state 原形，他大概像这样：

```javascript
{
  selectedSubreddit: 'frontend',
  postsBySubreddit: {
    frontend: {
      isFetching: true,
      didInvalidate: false,
      items: []
    },
    reactjs: {
      isFetching: false,
      didInvalidate: false,
      lastUpdated: 1439478405547,
      items: [
        {
          id: 42,
          title: 'Confusion about Flux and Relay'
        },
        {
          id: 500,
          title: 'Creating a Simple Application Using React JS and Flux Architecture'
        }
      ]
    }
  }
}
```

有两处重要的地方：

- 我们单独存储子栏目的信息因此我们可以单独缓存每一个子栏目。当用户来回切换子栏目时切换，将会立即更新，不需要重新读取数据除非我们想这么做。不需要担心会耗费大量内存：除非你保存了上万条项目并且用户很少关闭这个选项卡，否则你不需要做任何形式的清理。
- 对于每一个项目的列表，你可以保存 `isFetching` 字段现实加载界面，`didInvalidate` 可以在稍后数据陈旧时切换他，`lastUpdated` 可以知道数据最后的获取时间，和项目本身。在真实的应用程序，你也想保存分页状态例如 `fetchedPageCount` 和 `nextPageUrl`。

### 内嵌的 Entities

在这个例子中，我们存储了接收到的项和分页信息。不过，在有内嵌的的 entitie 彼此之间引用时不会很好的工作，或者你允许用户编辑项目。想象一下用户想编辑接收到的文章，但该文章复制到了 state tree 的几个地方。如果要实现这样的功能是痛苦的。

如果你有内嵌 entities，或你让用户编辑接收到的 entitie，你应该保持他们分别保存在 state，就像数据库一样。对于分页信息，你应该只通过他们的 ID 引用。这样让你可以使他们保存最新。`real world example` 演示了具体方法，通过使用 `normalizr` 使内嵌的 API 响应标准化。用了这种方法，你的 state 看起来会是这样的：

```javascript
{
  selectedSubreddit: 'frontend',
  entities: {
    users: {
      2: {
        id: 2,
        name: 'Andrew'
      }
    },
    posts: {
      42: {
        id: 42,
        title: 'Confusion about Flux and Relay',
        author: 2
      },
      100: {
        id: 100,
        title: 'Creating a Simple Application Using React JS and Flux Architecture',
        author: 2
      }
    }
  },
  postsBySubreddit: {
    frontend: {
      isFetching: true,
      didInvalidate: false,
      items: []
    },
    reactjs: {
      isFetching: false,
      didInvalidate: false,
      lastUpdated: 1439478405547,
      items: [ 42, 100 ]
    }
  }
}
```

在这个指南中我们不会使用标准化的 entitie，但有时你需要多为动态程序考虑。

## 处理 action

在 dispatch action 与网络请求结合的细节实现之前，我们先来为这些 action 创建 reducer。

`reducers.js`

```javascript
import { combineReducers } from 'redux'
import {
  SELECT_SUBREDDIT, INVALIDATE_SUBREDDIT,
  REQUEST_POSTS, RECEIVE_POSTS
} from '../actions'

function selectedSubreddit(state = 'reactjs', action) {
  switch (action.type) {
    case SELECT_SUBREDDIT:
      return action.subreddit
    default:
      return state
  }
}

function posts(state = {
  isFetching: false,
  didInvalidate: false,
  items: []
}, action) {
  switch (action.type) {
    case INVALIDATE_SUBREDDIT:
      return Object.assign({}, state, {
        didInvalidate: true
      })
    case REQUEST_POSTS:
      return Object.assign({}, state, {
        isFetching: true,
        didInvalidate: false
      })
    case RECEIVE_POSTS:
      return Object.assign({}, state, {
        isFetching: false,
        didInvalidate: false,
        items: action.posts,
        lastUpdated: action.receivedAt
      })
    default:
      return state
  }
}

function postsBySubreddit(state = {}, action) {
  switch (action.type) {
    case INVALIDATE_SUBREDDIT:
    case RECEIVE_POSTS:
    case REQUEST_POSTS:
      return Object.assign({}, state, {
        [action.subreddit]: posts(state[action.subreddit], action)
      })
    default:
      return state
  }
}

const rootReducer = combineReducers({
  postsBySubreddit,
  selectedSubreddit
})

export default rootReducer
```

这些代码中，有两个有趣的部分：

    - 我们使用了 ES6 的属性计算语法因此我们可以更新 `state[action.subreddit]` 和调用 `Object.assign()` 用一种简洁的方式。

        ```javascript
        return Object.assign({}, state, {
            [action.subreddit]: posts(state[action.subreddit], action)
        })
        ```

        等价于：

        ```javascript
        let nextState = {}
        nextState[action.subreddit] = posts(state[action.subreddit], action)
        return Object.assign({}, state, nextState)
        ```

    - 我们提取了 `posts(state,action)` 管理特定文章列表的 state。这是一个 [reducer composition](。。)！我们用他来选择如何分割 reducer 到更小的 reducers，在这个例子中，我们更新的 object 内的 item 到 `posts` reducer。在 [real world example](。) 会更进一步，展示如何为参数化分页创建 reducer factory。

请记住 reducer 只是一个函数，因此你可以使用函数式组合和高级别的函数，只要你用得顺畅。

## Async Action Creators

最后，应该如何使用我们之前创建的同步的 action craetor 配合网络请求使用？Redux 中标准的方式是使用 [Redux Thunk middleware](。)。他对应的 package 名称是 `redux-thunk`。我们后面会分析 moddleware 实现原理。现在，一件重要的事你需要明白：通过使用 middleware，action creator 可以返回 function 代替 action object。要使用这种方式，action creator 需要变为 [thunk](。)。

当 action creator 返回 function 时，该 function 会被 Redux Thunk middleware 执行。该 function 不需要是纯的；因此他可以带有其他操作，包括执行异步 API 调用。该 function 也可以 dispatch action － 例如我们早前定义的同步 action。

我们仍然可以定义这些特定的 thunk action creator 在我们的 `actions.js` 文件内：

`action.js`

```javascript
import fetch from 'isomorphic-fetch'

export const REQUEST_POSTS = 'REQUEST_POSTS'
function requestPosts(subreddit) {
  return {
    type: REQUEST_POSTS,
    subreddit
  }
}

export const RECEIVE_POSTS = 'RECEIVE_POSTS'
function receivePosts(subreddit, json) {
  return {
    type: RECEIVE_POSTS,
    subreddit,
    posts: json.data.children.map(child => child.data),
    receivedAt: Date.now()
  }
}

// Meet our first thunk action creator!
// Though its insides are different, you would use it just like any other action creator:
// store.dispatch(fetchPosts('reactjs'))

export function fetchPosts(subreddit) {

  // Thunk middleware knows how to handle functions.
  // It passes the dispatch method as an argument to the function,
  // thus making it able to dispatch actions itself.

  return function (dispatch) {

    // First dispatch: the app state is updated to inform
    // that the API call is starting.

    dispatch(requestPosts(subreddit))

    // The function called by the thunk middleware can return a value,
    // that is passed on as the return value of the dispatch method.

    // In this case, we return a promise to wait for.
    // This is not required by thunk middleware, but it is convenient for us.

    return fetch(`http://www.reddit.com/r/${subreddit}.json`)
      .then(response => response.json())
      .then(json =>

        // We can dispatch many times!
        // Here, we update the app state with the results of the API call.

        dispatch(receivePosts(subreddit, json))
      )

      // In a real world app, you also want to
      // catch any error in the network call.
  }
}
```

 > `fetch` 注意事项
>
> 这个例子中我们使用了 [`fetch` API](。)。他是最新的网络请求 API，用来替换 `XMLHttpRequest` 。因为大部分浏览器原生不支持，我们建议你使用 [isomorphic-fetch](。) 库：
>
> ```javascript
> // Do this in every file where you use `fetch`
> import fetch from 'isomorphic-fetch'
> ```
>
> 这个库内部使用了 [`whatwg-fetch` polyfill](。)，服务端使用了 [node-fetch](。)，因此如果你的应用程序是[通用](。)的你不需要更改 API 的调用方式。
>
> 注意任意的 `fetch` polyfill 都假设 [Promise](。) polyfille 已经存在。最简单的方式确保你拥有 Promise polyfill 是启用 Babel 的 ES6 polyfill
>
> ```javascript
> // Do this once before any other code in your app
> import 'babel-polyfill'
> ```

如何将 Redux Thunk middleware 包含到 dispatch 机制中？我们使用 Redux 提供的 `applyMiddleware()` 增强 store 功能，如下面所示：

`index.js`

```javascript
import thunkMiddleware from 'redux-thunk'
import createLogger from 'redux-logger'
import { createStore, applyMiddleware } from 'redux'
import { selectSubreddit, fetchPosts } from './actions'
import rootReducer from './reducers'

const loggerMiddleware = createLogger()

const store = createStore(
  rootReducer,
  applyMiddleware(
    thunkMiddleware, // lets us dispatch() functions
    loggerMiddleware // neat middleware that logs actions
  )
)

store.dispatch(selectSubreddit('reactjs'))
store.dispatch(fetchPosts('reactjs')).then(() =>
  console.log(store.getState())
)
```

thunk 的一个好处是他可以在彼此之间 dispatch：

`action.js`

```javascript
import fetch from 'isomorphic-fetch'

export const REQUEST_POSTS = 'REQUEST_POSTS'
function requestPosts(subreddit) {
  return {
    type: REQUEST_POSTS,
    subreddit
  }
}

export const RECEIVE_POSTS = 'RECEIVE_POSTS'
function receivePosts(subreddit, json) {
  return {
    type: RECEIVE_POSTS,
    subreddit,
    posts: json.data.children.map(child => child.data),
    receivedAt: Date.now()
  }
}

function fetchPosts(subreddit) {
  return dispatch => {
    dispatch(requestPosts(subreddit))
    return fetch(`http://www.reddit.com/r/${subreddit}.json`)
      .then(response => response.json())
      .then(json => dispatch(receivePosts(subreddit, json)))
  }
}

function shouldFetchPosts(state, subreddit) {
  const posts = state.postsBySubreddit[subreddit]
  if (!posts) {
    return true
  } else if (posts.isFetching) {
    return false
  } else {
    return posts.didInvalidate
  }
}

export function fetchPostsIfNeeded(subreddit) {

  // Note that the function also receives getState()
  // which lets you choose what to dispatch next.

  // This is useful for avoiding a network request if
  // a cached value is already available.

  return (dispatch, getState) => {
    if (shouldFetchPosts(getState(), subreddit)) {
      // Dispatch a thunk from thunk!
      return dispatch(fetchPosts(subreddit))
    } else {
      // Let the calling code know there's nothing to wait for.
      return Promise.resolve()
    }
  }
}
```

我们可以逐渐的写复杂的异步控制流程，这些代码都非常相似：

```javascript
store.dispatch(fetchPostsIfNeeded('reactjs')).then(() =>
  console.log(store.getState())
)
```

> 关于服务器渲染的说明
>
> Async action craetor 对于服务端渲染非常方便。你可以创建 stroe，dispatch 一个单独的 async 用来 dispatch 其他 async action cretor 以捕捉应用程序整个章节的数据，并只渲染 Promise 返回的完整内容。然后你的 store 在渲染之前已经与 state 组合。

[Thunk Middlesware](。) 不是 Redux 中唯一处理异步 action 的方式。你可以使用 [redux-promise](。) 或 [redux-promise-middleware](。) 进行 dispatch Promise 替代 function。你可以使用 [redux-rx](。) dispatch Observables。你甚至可以写自定义的 middleware 描述调用 API 的方式，例如 [real world example](。) 所做的那样。你可以进行一些尝试，选择你喜欢的方式，并按照它实现，无论是否使用 middleware。

## 连接到 UI

dispatch async action 与 dispatch synchronous action 没有区别，因此我们不在此讨论细节。见 [Usage with React](。)  介绍如何与 React 协同使用。见 [Example:Reddit API](。) 有本章节讨论的完整代码。

## 下一步

阅读 [Async Flow](。) 概括了 async action 如何适应 Redux flow。
