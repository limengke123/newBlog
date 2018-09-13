---
title: react使用指南
date: 2018-09-12 13:37:15
tags:
  - react
categories: 
  - 前端
---

# react使用指南

使用 `react` 有段时间了，总感觉用的不够深入，连最基本异步处理方案 `redux-saga` 也才是前端时间刚学的。鉴于此，在 `github` 上搜了下相关的 `react` 项目，找到了一个外国人写的[一个项目](https://github.com/r-park/soundcloud-redux)，看了内部 `react` 以及一些库的使用，整个 `react` 生态用的很不错，很多地方我都没有接触过，所以对一些写法和库的使用上做一些记录和总结。

## 相关类库

由于查看的那个项目是 *2017年* 写的，所以有一些库不太一样了，这里我结合自身的使用情况总结下一个还算完整的 `react` 项目可能会用到库：

| 库名 | 用途 | 类似功能的库 |
| --- | --- | --- |
| `react` | 核心库 | / |
| `react-dom` | 核心库 | / |
| `prop-types` | props校验库 | / |
| `react-router-dom` | 路由库 | `reach/router` |
| `redux` | 状态管理库 | `Mobx`、`rematch` |
| `react-redux` | 连接 `react`、`redux` | / |
| `redux-saga` | `redux` 中间件，解决异步问题 | `redux-thunk`、`redux-promise` |
| `redux-devtools-extension` | `chrome` 的 `redux` 调试工具 | / |
| `reselect` | `store` 上取值能够缓存 | / |
| `immutable` | 不可变数据 | / |

由此可见，`react` 全家桶一次性学习下来，还是有一定的门槛的，接下来汇总下基本使用套路。

## 使用套路

老实说，`react` 是一个学习、使用相当平滑的库，所以简单的使用还是比较容易的，主要学习的难点还是在 `redux` 以及像 `immutable` 这样的很少用的库。之前，我是没有用过`immutable` 和 `reselect` ，这里就对着别人项目记录下。

### redux初始化

`redux` 本身是一个很纯粹的状态管理库，和 `react` 本身没有任何瓜葛，但是用 `react-redux` 可以把 `react` 和 `redux` 结合起来。具体细节 `api` 不谈，直接记录平时如何使用：

```js
import React from 'react'
import ReactDOM from 'react-dom'
import { createStore, applyMiddleware } from 'redux'
import {Provider} from 'react-redux'
import createSagaMiddleware from 'redux-saga'
import {composeWithDevTools} from 'redux-devtools-extension'
import reducer from './store/reducers'
import rootSaga from './store/sagas/index'
import { AppWithRouter } from './router/router'
const sagaMiddleware = createSagaMiddleware()
const composeEnhancers = composeWithDevTools({})

const store = createStore(
  reducer,
  composeEnhancers(
      applyMiddleware(sagaMiddleware)
  ),
)

sagaMiddleware.run(rootSaga)

ReactDOM.render(
    <Provider store={store}>
      <AppWithRouter />
    </Provider>,
    document.getElementById('root')
  )
```

这里根据 `reducer` 生成了 `store`，把 `store` 挂载到 `Provider` 上面去了，后面的子组件就会根据 `context` 去拿到 `store` 上的值。

这里的 `AppWithRouter` 是我想要渲染的组件，`reducer` 、`rootSaga` 是我业务相关的内容，而其他内容可以发现，基本都是固定的，下一个项目基本可以照搬过来。

### reducer写法

这里先看我的很一般的 `reducer` 写法，再看一下别人结合 `immutable` 的 `reducer`。

```js
// 我的reducer写法
import {actionTypes} from '../action-type'
export function pageList(state = {
    list: [],
    isLoading: false
}, action) {
    switch (action.type) {
        case actionTypes.FETCH_LIST_SUCCESS:
            return {
                ...state,
                list: action.payload
            }
        case actionTypes.LIST_LOADING:
            return {
                ...state,
                isLoading: action.payload
            }
        default:
            return state
    }
}
```

```js
// action-type.js
export const actionTypes = {
    // 详情页面
    FETCH_DETAIL: 'FETCH_DETAIL',
    FETCH_DETAIL_SUCCESS: 'FETCH_DETAIL_SUCCESS',
    DETAIL_LOADING: 'DETAIL_LOADING',

    // 列表页面
    FETCH_LIST_SUCCESS: 'FETCH_LIST_SUCCESS',
    LIST_LOADING: 'LIST_LOADING',
    FETCH_LIST: 'FETCH_LIST',

    // tab
    CHANGE_TAB: 'CHANGE_TAB',

    // currentPage
    CHANGE_PAGE: 'CHANGE_PAGE'
}
```

只要注意把固定的字符串全部写成变量。

由于 `redux` 需要保持纯函数的特点，所以 `redux` 是不能直接修改 `state` 的值，应该返回一个全新的 `state` ，所以如果 `state` 的嵌套层数很深的话，要返回全新的 `state` 就比较麻烦了，所以这里就引申出来 `immutable`，同样在组件 `shouldComponentUpdate` 时需要对比两个对象时，`immutable` 也能帮上很大的忙。

看看别人 `reducer` 的用法：

```js
import { Record } from 'immutable';
import { searchActions } from './actions';


export const SearchState = new Record({
  currentQuery: null,
  open: false
});


export function searchReducer(state = new SearchState(), {payload, type}) {
  switch (type) {
    case searchActions.LOAD_SEARCH_RESULTS:
      return state.merge({
        open: false,
        currentQuery: payload.query
      });

    case searchActions.TOGGLE_SEARCH_FIELD:
      return state.set('open', !state.open);

    default:
      return state;
  }
}
```

相比我的直接对象，这里用了 `immutable` 的 `Record`，在 `reducer` 内部需要修改 `state` 的时候，直接调用 `set` 方法就去修改了，在层级很深的对象的时候是非常方便的。

### saga的写法

当初有点恐惧学习 `redux-saga`，实际去学习和使用的时候发现还是很不错的，相比`redux-thunk` 去强行让 `action` 能够是个函数，`redux-saga` 还是保持 `action` 是一个对象，脏活累活全丢给 `saga` 去做，`redux` 的那一块逻辑依然保持之前一样纯净。先上例子：

```js
import { call, fork, select, take, takeLatest } from 'redux-saga/effects';
import { fetchSearchResults } from 'src/core/api';
import history from 'src/core/history';
import { getTracklistById } from 'src/core/tracklists';
import { searchActions } from './actions';


export function* loadSearchResults({payload}) {
  const { query, tracklistId } = payload;
  const tracklist = yield select(getTracklistById, tracklistId);
  if (tracklist && tracklist.isNew) {
    yield call(fetchSearchResults, tracklistId, query);
  }
}


//=====================================
//  WATCHERS
//-------------------------------------

export function* watchLoadSearchResults() {
  yield takeLatest(searchActions.LOAD_SEARCH_RESULTS, loadSearchResults);
}

export function* watchNavigateToSearch() {
  while (true) {
    const { payload } = yield take(searchActions.NAVIGATE_TO_SEARCH);
    yield history.push(payload);
  }
}


//=====================================
//  ROOT
//-------------------------------------

export const searchSagas = [
  fork(watchLoadSearchResults),
  fork(watchNavigateToSearch)
];
```

这玩意按我目前的理解，saga分两块，一块专门用来 `watch`，一块是处理，`watch` 用`while` 死循环 `take` 或者`takeEvery`、`takeLatest` 去 `watch` 对应的 `action/type`， 然后调用另一个 `sagas` ，在另一个 `sagas` 用 `call` 之类的去调用异步的 `api/service`。

### store和视图

扯了半天 `redux`，看最后是怎么把 `redux` 上的store数据关联到视图层上，以及视图如何去改变store里面的值，主要还是 `react-redux` 用 `connect` 把 `store` 的数据以及 `dispatch` 给组件，这样组件就能获取数据以及修改数据了。

先看我的常规做法：

```js
import React from 'react'
import {compose} from 'redux'
import {withRouter} from 'react-router-dom'
import {connect} from 'react-redux'
import {actionTypes} from '../store/action-type'

class DetailPage extends React.Component {
    componentDidMount () {
        const {fetchDetail} = this.props
        fetchDetail()
    }
    render () {
        const {detail, isLoading} = this.props
        return (
           xxx
        )
    }
}

const mapStateToProps = state => {
    return {
        detail: state.detailData.data,
        isLoading: state.detailData.isLoading
    }
}

const mapDispatchToProps = (dispatch, ownProps) => {
    return {
        fetchDetail() {
            dispatch({
                type: actionTypes.FETCH_DETAIL,
                payload: ownProps.match.params.id
            })
        }
    }
}

export default compose(
    withRouter,
    connect(mapStateToProps, mapDispatchToProps),
)(DetailPage)
```

还是很简单的，直接在 `connect` 中传两个参数 `mapStateToProps`、`mapDispatchToProps` 过去就完事了，这样组件需要什么值，需要什么方法都能提供。

顺带一提，用上 `withRouter`，这样路由信息也能给到组件。

看下用了 `reselect` 之后，是怎么用的：

```js
import React from 'react';
import { connect } from 'react-redux';
import { createSelector } from 'reselect';
import classNames from 'classnames';
import { List } from 'immutable';
import PropTypes from 'prop-types';
import { getBrowserMedia, infiniteScroll } from 'src/core/browser';
import { audio, getPlayerIsPlaying, getPlayerTrackId, playerActions } from 'src/core/player';
import { getCurrentTracklist, getTracksForCurrentTracklist, tracklistActions } from 'src/core/tracklists';

export class Tracklist extends React.Component {
  static propTypes = {
    compactLayout: PropTypes.bool,
    displayLoadingIndicator: PropTypes.bool.isRequired,
    isMediaLarge: PropTypes.bool.isRequired,
    isPlaying: PropTypes.bool.isRequired,
    loadNextTracks: PropTypes.func.isRequired,
    pause: PropTypes.func.isRequired,
    pauseInfiniteScroll: PropTypes.bool.isRequired,
    play: PropTypes.func.isRequired,
    selectTrack: PropTypes.func.isRequired,
    selectedTrackId: PropTypes.number,
    tracklistId: PropTypes.string.isRequired,
    tracks: PropTypes.instanceOf(List).isRequired
  };

  componentDidMount() {
    infiniteScroll.start(
      this.props.loadNextTracks,
      this.props.pauseInfiniteScroll
    );
  }

  componentWillUpdate(nextProps) {
    if (nextProps.pauseInfiniteScroll !== this.props.pauseInfiniteScroll) {
      if (nextProps.pauseInfiniteScroll) {
        infiniteScroll.pause();
      }
      else {
        infiniteScroll.resume();
      }
    }
  }

  componentWillUnmount() {
    infiniteScroll.end();
  }

  render() {
    const { compactLayout, isMediaLarge, isPlaying, pause, play, selectedTrackId, selectTrack, tracklistId, tracks } = this.props;

    return (
      xxxx
    );
  }
}


//=====================================
//  CONNECT
//-------------------------------------

const mapStateToProps = createSelector(
  getBrowserMedia,
  getPlayerIsPlaying,
  getPlayerTrackId,
  getCurrentTracklist,
  getTracksForCurrentTracklist,
  (media, isPlaying, playerTrackId, tracklist, tracks) => ({
    displayLoadingIndicator: tracklist.isPending || tracklist.hasNextPage,
    isMediaLarge: !!media.large,
    isPlaying,
    pause: audio.pause,
    pauseInfiniteScroll: tracklist.isPending || !tracklist.hasNextPage,
    play: audio.play,
    selectedTrackId: playerTrackId,
    tracklistId: tracklist.id,
    tracks
  })
);

const mapDispatchToProps = {
  loadNextTracks: tracklistActions.loadNextTracks,
  selectTrack: playerActions.playSelectedTrack
};

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(Tracklist);

```

```js
// selector.js
export function getBrowserMedia(state) {
  return state.browser.media;
}

```

具体关注下用了 `reselect` 之后，`mapStateToProps` 和我之前的写法发生了变化，正如给的例子那样用 `createSelector` 包了一层，同时传入两个参数进去，第一个参数是个从 `state` 上取值的函数，就像上面的 `getBrowserMedia` 这个例子一样。至于 `mapDispatchToProps` 的写法，在我的用法是写一个接受 `dispatch` 的函数同时返回一个对象，当然也可以像上面一样传入一个对象，这个对象 `redux` 就默认做为 `action`。

### props 验证

上面介绍了那么多 `redux` 相关写法，`redux` 确实算是 `react` 学习上的一个难点，现在讲点轻松点的。`redux` 推崇**容器组件**和**展示组件**，实际上在写 `react` 应用的时候，你也可能不太会注意到，其实用 `connect` 这个高阶函数包装过的组件就是所谓的**容器组件**，而传给 `connect` 的组件，其实就是我们写的**展示组件**，写的多了就会发现哈，我们越来越少地用到了组件内部的 `state` 去控制组件，反而大部分情况都是直接用 `props` 去控制组件，这也得益于 `redux` 能够提供类似全局变量 `store` 的取值和改变值的方式。所以说回来，对于一个 `react` 组件而言，`state` 对应内部状态，`props` 对应外部传入值，`props` 由于 `redux` 等状态管理库盛行，使用频率也大幅增加，所以我们需要严格要求好外部传入的 `props`的类型要符合组件规定的。`prop-types` 就是解决这个问题的，当然你也可以不去校验 `props` 的类型。

```js
import React from 'react'
export default class Test extends React.Component {
    static propTypes = {
        compactLayout: PropTypes.bool,
        displayLoadingIndicator: PropTypes.bool.isRequired,
        isMediaLarge: PropTypes.bool.isRequired,
        isPlaying: PropTypes.bool.isRequired,
        loadNextTracks: PropTypes.func.isRequired,
        pause: PropTypes.func.isRequired,
        pauseInfiniteScroll: PropTypes.bool.isRequired,
        play: PropTypes.func.isRequired,
        selectTrack: PropTypes.func.isRequired,
        selectedTrackId: PropTypes.number,
        tracklistId: PropTypes.string.isRequired,
        tracks: PropTypes.instanceOf(List).isRequired
    };
    static defaultProps = {
        compactLayout: true
    }
    render () {
        return xxx
    }
}

```

## 总结

`react` 本身不难，甚至我觉得比起 `vue` 而言更为简单，使用难点主要还是在于一些第三方库的搭配使用，所以本文也是基于这个点，记录下一些 `react` 常见用法，以便日后忘记了可以翻阅。

---

![Olive Trees](react使用指南/1755523965.jpg)

> Vincent van Gogh – Olive Trees  1888