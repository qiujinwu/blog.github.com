---
title: antd随笔
date: 2017-12-02 23:20:52
tags:
 - antd
categories:
 - 学习笔记
---


antd <https://ant.design/index-cn>

# 环境

本着学习的目的，最开始自己从零折腾，不过后来放弃了，学习从<https://github.com/zuiidea/antd-admin>开始

可以直接删除业务/页面代码，保留配置继续开发，后者体验网址<http://antd-admin.zuiidea.com>

# html5/es6+/react
大量使用es6语法

简化的import/export，以及[大括号语法](https://segmentfault.com/q/1010000005762199)，自动导出子模块（变量）
``` js
import {request, config} from 'utils'
const {api} = config;
```
[异步函数](http://www.ruanyifeng.com/blog/2015/05/async.html)/`async`
``` js
export async function loginCb(data) {
  return request({
    url: api.userLoginCB,
    method: 'get',
    data,
  })
}
```
[生成器函数](http://www.ruanyifeng.com/blog/2015/04/generator.html)/`yield`/`*函数`
``` js
yield put({
  type: 'querySuccess',
  payload: {
    data: data,
  },
})

* query({
          url,
          payload,
        }, {call, put}) {
  const {data} = yield call(url, payload);
  yield put({
    type: 'querySuccess',
    payload: {
      data: data,
    },
  })
},
```

箭头函数
``` js
export default connect(({ detail, loading }) => ({ state:detail, loading }))(Detail)

const handleOk = () => {
  validateFields((errors) => {
    if (errors) {
      return
    }
    const data = {
      ...item,
      ...getFieldsValue(),
    };
    onOk(data)
  })
};

// 多重
const handleChange = (count) => (fileList) => {
	// do sth
};
```
合并对象/...
``` js
const data = {
  ...item,
  ...getFieldsValue(),
};
```

const/let声明变量/常亮

(不)等于 ===/!==

[map/filter](https://www.cnblogs.com/zxyun/p/7019631.html)

window.localStorage保存本地配置

## React
react组件有[三种声明](https://www.cnblogs.com/wonyun/p/5930333.html)方式

antd大量使用`函数式定义`的写法，这种写法是废代码不多，但是不支持状态和react事件

``` js
import React from 'react'
import { config } from 'utils'
import styles from './Footer.less'

const Footer = () => (<div className={styles.footer}>
  {config.footerText}
</div>)

export default Footer
```

es5原生方式React.createClass定义的组件

es6形式的extends React.Component定义的组件

# 跳转
跳转可以用history接口<https://developer.mozilla.org/en-US/docs/Web/API/History_API>

或者更高层的history库 <https://github.com/ReactTraining/history>

不过推荐dispatch一个routerRedux异步任务的方式

``` js
dispatch(routerRedux.push({
  pathname: buildUrl(route.apartmentBatchPrice,{id:record.id}),
}))
```
## React router的迁移问题

React router 2到4接口改变了很多，antd/dva版本上也有出现很多问题，参见<https://github.com/gmfe/Think/issues/6>

新版本不再有query成员，不过可以手动处理下
``` js
const queryString = require('query-string');
location.query = queryString.parse(location.search);
```

## 参数
跳转时，一般使用url的query参数传递参数，不过若使用[ReactTraining/history](https://github.com/ReactTraining/history) 库，可以借助`location.state`

1. location.pathname - The path of the URL
1. location.search - The URL query string
1. location.hash - The URL hash fragment
1. location.state - Some extra state for this location that does not reside in the URL (supported in createBrowserHistory and createMemoryHistory)
2. location.key - A unique string representing this location (supported in createBrowserHistory and createMemoryHistory)

> routerRedux.push的参数就是一个location对象

# [Model复用](https://github.com/dvajs/dva-model-extend)
借助dva提供的`modelExtend`，可以复用model的变量和方法，

``` js
import modelExtend from 'dva-model-extend'

const model = {
  reducers: {
    updateState (state, { payload }) {
      return {
        ...state,
        ...payload,
      }
    },
  },
};

const pageModel = modelExtend(model, {
  state: {
    list: [],
    pagination: {
      showSizeChanger: false,
      showQuickJumper: false,
      showTotal: total => `共 ${total} 条记录`,
      current: 1,
      total: 0,
    },
  },

  reducers: {
    querySuccess (state, { payload }) {
      const { list, pagination } = payload
      return {
        ...state,
        ...{list:list},
        pagination: {
          ...state.pagination,
          ...pagination,
        },
      }
    },
  },

});


module.exports = {
  model,
  pageModel,
};
```

和C++的继承有几分相似的是，modelExtend的继承可以继续继承，也可以多继承

``` js
const pageModel = modelExtend(model, model2 {
	// ...
}
```

# service优化
service定义了调用后端Api的接口，每个route的service结构都大同小异，考虑到常用的就CRUD，对于同一个后端，使用的http method/参数定义也相同，所以可以抽象以下。

下面的代码架设了create/query/detail/update/enable/disable/remove等常用方法的请求格式
``` js
import {request, config} from 'utils'
import {buildUrl} from '../utils/url'

export function createService(keys) {
  let res = {};
  let exclude = {}
  if (keys["create"]) {
    res["create"] =
      async function create(data, param=null) {
        return request({
          url: buildUrl(keys.create, param),
          method: 'post',
          data,
        })
      }
    exclude["create"] = null
  }

  if (keys["query"]) {
    res["query"] =
      async function query(data, param=null) {
        return request({
          url: buildUrl(keys.query, param),
          method: 'get',
          data,
        })
      }
    exclude["query"] = null
  }

  if (keys["detail"]) {
    res["detail"] =
      async function detail(param = null) {
        return request({
          url: buildUrl(keys.detail, param),
          method: 'get',
        })
      }
    exclude["detail"] = null
  }

  if (keys["update"]) {
    res["update"] =
      async function update(data, param=null) {
        return request({
          url: buildUrl(keys.update, param),
          method: 'put',
          data: data,
        })
      }
    exclude["update"] = null
  }

  if (keys["enable"]) {
    res["enable"] =
      async function enable(param = null) {
        return request({
          url: buildUrl(keys.enable, param),
          method: 'post',
        })
      }
    exclude["enable"] = null
  }

  if (keys["disable"]) {
    res["disable"] =
      async function disable(param = null) {
        return request({
          url: buildUrl(keys.disable, param),
          method: 'post',
        })
      }
    exclude["disable"] = null
  }

  if (keys["remove"]) {
    res["remove"] =
      async function remove(param = null) {
        return request({
          url: buildUrl(keys.remove, param),
          method: 'delete',
        })
      }
    exclude["remove"] = null
  }

  const methods = {
    "post": null,
    "put": null,
    "get": null,
    "delete": null,
    "patch": null,
  }

  for (const key in keys) {
    if (keys.hasOwnProperty(key)){
      // 排除上面的通用方法
      if(key in exclude) continue
      // 必须是对象
      if(typeof keys[key] === 'object'){
        const rt = keys[key]
        // 合法的方法
        if(!rt.method in methods) {
          continue
        }

        res[key] =
          async function create(data, param=null) {
            return request({
              url: buildUrl(rt.url, param),
              method: rt.method,
              data,
            })
          }
      }
    }
  }

  return res
}
```
``` js
import {request, config} from 'utils'
import {createService} from "./common"

exports.services = createService(config.api.user);
```
``` js
user: {
  "query": `${APIV1}/users/`,
  "update": `${APIV1}/users/:id`,
  "detail": `${APIV1}/users/:id`,
  "enable": `${APIV1}/users/:id/enable`,
  "disable": `${APIV1}/users/:id/disable`
},
```

一些特殊的方法，则简单描述即可

``` js
import {request, config} from 'utils'
import {createService} from "./common"

let routes = config.api.price
if(typeof routes["generate"] != 'object'){
  routes['generate'] = {
    "method": "post",
    "url": routes['generate']
  }
}

exports.services = createService(routes);
```
``` js
price: {
  "create": `${APIV1}/:id/prices`,
  "update": `${APIV1}/:id/prices`,
  "query": `${APIV1}/:id/prices`,
  "generate": `${APIV1}/:id/prices/generate`,
}
```


# Url
大量使用[path-to-regexp](https://github.com/pillarjs/path-to-regexp)

有用到几种场景

判断是否是我们需要的url
``` js
setup({dispatch, history}) {
  history.listen((location) => {

    const match = pathToRegexp(route.apartments).exec(location.pathname);
    if (match) {
      location.query = queryString.parse(location.search);
      const payload = location.query || {page: 1, count: countPePage};
      dispatch({
        type: 'query',
        payload,
      })
    }
  })
},
```

构建url用于跳转，解析url中的参数
``` js
import pathToRegexp from 'path-to-regexp'

module.exports = {
  buildUrl:(url, param) => {
    if(!param) return url;

    const toPath = pathToRegexp.compile(url);
    return toPath(param);
  },

  parseUrl:(url,route) => {
    const match = pathToRegexp(route).exec(url)
    if(!match) return {}

    let urlDatas = {};
    let index = 1;
    const keys = pathToRegexp.parse(route)
    keys.forEach(function(item){
      if(typeof item !== 'object'){
        return
      }
      urlDatas[item.name] = match[index]
      index ++
    });
    return urlDatas
  }
};
```
``` js
yield put(routerRedux.push({
  pathname: buildUrl(route.storeDetailIndex,{id:store.id,tab:route.storeDetailTab.rule}),
}))
```
``` js
const urlDatas = parseUrl(location.pathname, current.route);
pathArray.forEach(function (item) {
  if(item === current || !item.route) return;
  item.route2 = buildUrl(item.route,urlDatas)
})
```

# [dva-loading](https://github.com/dvajs/dva/tree/master/packages/dva-loading)

通过绑定一个[redux-saga](https://github.com/redux-saga/redux-saga)异步请求来实现UI的`加载中`的效果

关于dva以及react全家桶的一些概念，参见<https://github.com/dvajs/dva/blob/master/README_zh-CN.md>

``` js
import createLoading from 'dva-loading'
// 1. Initialize
const app = dva({
  ...createLoading({
    effects: true,
  }),
});
```
``` js
export default connect(({ detail, loading }) => ({ state:detail, loading }))(Detail)
```
``` js
<Button type="primary" size="large" loading={loading.effects.login}>
  登录
</Button>
```

antd很多组件都集成了loading属性。例如[button](https://ant.design/components/button-cn/#API)，[Table](https://ant.design/components/table-cn/#Table)

当页面不存在这些支持loading的控件，或者不合适使用时，antd-admin提供了一个全局的[Loader](https://github.com/zuiidea/antd-admin/tree/master/src/components/Loader)控件
``` js
const LoginCB = ({location, dispatch, loading}) => {
  return (
    <div>
      <Loader fullScreen spinning={loading.effects['login_cb/login']} />
    </div>
  )
}
```

# 富文本编辑器
官方推荐的[react-lz-editor](https://github.com/leejaen/react-lz-editor)相当不错，并且支持markdown和普通的[wysiwyg](https://baike.baidu.com/item/%E6%89%80%E8%A7%81%E5%8D%B3%E6%89%80%E5%BE%97/8721378?fr=aladdin&fromid=190526&fromtitle=WYSIWYG)模式，不过有一个需求（`粘贴富文本格式`)似乎满足不了,

[react-draft-wysiwyg](https://github.com/jpuri/react-draft-wysiwyg)也是个不错的选择
``` js
import {Editor} from 'react-draft-wysiwyg';
import 'react-draft-wysiwyg/dist/react-draft-wysiwyg.css';
import { EditorState, convertToRaw, ContentState, convertFromHTML } from 'draft-js';

const checkContent = (rule, value, callback) => {
  if (value && state.editorState) {
    const content = draftToHtml(convertToRaw(state.editorState.getCurrentContent()));
    if(content.length < 9 || content.length >= 1024000){
      callback("content must be between 2 and 1024000 characters\n");
      return
    }
  }
  callback();
};

const uploadImageCallBack = (file) => {
  return new Promise(
    (resolve, reject) => {
      const xhr = new XMLHttpRequest();
      xhr.open('POST', '/api/v1/upload');
      const data = new FormData();
      data.append('file', file);
      xhr.send(data);
      xhr.addEventListener('load', () => {
        let response = JSON.parse(xhr.responseText);
        response = { data: { link: response.data.url,alt: "fasdf" } }
        resolve(response);
      });
      xhr.addEventListener('error', () => {
        const error = JSON.parse(xhr.responseText);
        reject(error);
      });
    }
  );
};

const onEditorStateChange = (editorState) => {
  dispatch({
    type: `${pageKey}/updateState`,
    payload: {
      editorState
    }
  })
};

<Editor
  editorState={state.editorState}
  wrapperClassName="demo-wrapper"
  editorClassName="demo-editor"
  toolbar={{
    image: { uploadCallback: uploadImageCallBack, alt: { present: false, mandatory: false } },
  }}
  onEditorStateChange={onEditorStateChange}
/>
```

百度开源的[ueditor](http://ueditor.baidu.com/website/)非常完善,但是体积非常大，一个精简版[UMeditor](http://ueditor.baidu.com/website/umeditor.html)


# 一些不错的三方库
1. [prop-types](https://github.com/facebook/prop-types),检查组件参数
2. [path-to-regexp](https://github.com/pillarjs/path-to-regexp),url工具库
3. [react-helmet](https://github.com/nfl/react-helmet),设置title以及头部信息
4. [nprogress](https://github.com/rstacruz/nprogress),加载进度条，和dva-loading配合使用
5. [axios](https://github.com/axios/axios),基于promise的http客户端库，dva-admin实现了一个比较完善的[request函数](https://github.com/zuiidea/antd-admin/blob/master/src/utils/request.js)
6. [query-string](https://github.com/sindresorhus/query-string)，解析url查询参数
1. [react-amap](https://github.com/ElemeFE/react-amap)，高德地图
1. [sprintf.js](https://github.com/alexei/sprintf.js/),类似于C printf的占位符打印实现
1. [ReactInlineEdit](https://github.com/kaivi/ReactInlineEdit),双击编辑控件
1. [qs](https://github.com/ljharb/qs),A querystring parser with nesting support
1. [lodash](https://github.com/lodash/lodash),工具库
7. 其他<https://ant.design/docs/react/recommendation-cn>

``` js
if (lastHref !== href) {
  NProgress.start()
  if (!loading.global) {
    NProgress.done()
    lastHref = href
  }
}
```

# 日期
antd使用moment作为日期处理库

获取当月/指定月的区间

``` js
import Moment from 'moment';

const getMonthDateRange = (param) => {
  const startDate = Moment([param.year(), param.month()]);

  // Clone the value before .endOf()
  const endDate = Moment(startDate).endOf('month');

  // make sure to call toDate() for plain JavaScript date type
  return { from: startDate, to: endDate };
}

const getCurMonthRange = () => {
  return getMonthDateRange(Moment())
}

module.exports = {
  getMonthDateRange,
  getCurMonthRange,
}
```
