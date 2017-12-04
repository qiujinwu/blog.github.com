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

# [表单](https://ant.design/components/form-cn/)
``` js
<Form.Item {...props}>
  {children}
</Form.Item>
```

form基于[rc-form](https://github.com/react-component/form)，一些选项可以参考<https://github.com/react-component/form#option-object>

经过 `getFieldDecorator` 包装的控件，表单控件会自动添加 `value`（或 `valuePropName` 指定的其他属性） `onChange`（或 trigger 指定的其他属性），数据同步将被 Form 接管，这会导致以下结果：
1. 你不再需要也不应该用 `onChange` 来做同步，但还是可以继续监听 onChange 等事件。
1. 你不能用控件的 `value` `defaultValue` 等属性来设置表单域的值，默认值可以用 `getFieldDecorator` 里的 `initialValue`。
1. 你不应该用 `setState`，可以使用 `this.props.form.setFieldsValue` 来动态改变表单值。

## [自定义控件适配](https://github.com/ant-design/ant-design/issues/5700)
参考<https://ant.design/components/form/#components-form-demo-customized-form-controls>

关键两点
1. 初始值从`props.value`获取，这个值从`getFieldDecorator`的`initialValue`获取
2. 当控件`value`值改变时，通过`props.onChange`获取到触发`getFieldDecorator`的回调，并触发之

``` js
import React from 'react'
import {Select} from 'antd';
import {connect} from 'dva'
import {Icon} from 'antd';
import PropTypes from 'prop-types';
const {Option} = Select;

class IDSelect extends React.Component {
  constructor(props) {
    super(props);

    const value = this.props.value || [];
    this.state = {
      current_value: undefined,
      list: [],
    };
  }

  componentWillReceiveProps(nextProps) {
    // Should be a controlled component.
    if ('value' in nextProps) {
      // ----- mark------
      const value = nextProps.value;
      if(value === undefined) return

      if (!value){
        this.setState({current_value: null});
      }else{
        this.setState({current_value:String(value)});
      }
    }

    if ('list' in nextProps) {
      const list = nextProps.list;
      if(list === undefined) return
      this.setState({list});
    }
  }

  render() {
    const handleChange = (value) => {
      this.setState({current_value:value});
      // ----- mark------
      const onChange = this.props.onChange;
      if (onChange) {
        onChange(parseInt(value));
      }
    }

    const {list,value,...props} = this.props
    return (
      <span>
      {this.state.list.length > 0 && <Select
        {...props}
        defaultValue={this.state.current_value}
        onChange={handleChange}
        >
          {this.state.list.map(function (object, i) {
            return <Option value={String(object.id)} key={i}>{object.name}</Option>;
          })}
        </Select>}
      </span>
    );
  }
}

IDSelect.propTypes = {
  value: PropTypes.number,
  list: PropTypes.arrayOf(PropTypes.object),
};

module.exports = {
  IDSelect,
};
```
``` js
<FormItem>
  {getFieldDecorator('user_id', {
    initialValue: state.editType === 'create' ? null : data.user_id,
    rules: [
      {
        required: true,
      }
    ],
  })(
    <IDSelect
      disabled={state.editType !== 'create'}
      list={state.users}
    />
  )}
</FormItem>
```

## 提交

提交时，使用`form.validateFields`做校验，参数中`values`就是包含所有表单参数的map,map的key是`getFieldDecorator`的第一个参数。*例如上面的user_id*

某些控件返回的数据并非后端需要的格式（例如DatePicker返回一个moment，Select返回一个String类型的ID），在提交到后端的API之前，在这里做格式的转换

``` js
const handleSubmit = (e) => {
  e.preventDefault();
  form.validateFields((err, values) => {
    if (!err) {
        // do submit
    }
  });
};
```

## [校验](https://github.com/yiminghe/async-validator)
规则见<https://github.com/yiminghe/async-validator>

``` js
const checkCoordinate = (rule, value, callback) => {
  if(!parseCoordinate(value)){
    callback("错误的经纬度");
    return
  }

  callback();
};

<FormItem>
  {getFieldDecorator('coordinate', {
    initialValue: state.editType === 'create' ? "" : String(data.lng) + ',' + String(data.lat),
    rules: [
      {
        required: true,
        min: 1,
        max: 64,
      },
      {validator: checkCoordinate},
    ],
  })(<Input placeholder="请输入经纬度" style={{width: '100%'}}/>)}
</FormItem>
```
``` js
module.exports = {
  parseCoordinate:(value) => {
    const values = value.split(",");
    if (values.length !== 2) {
      return null;
    }

    try {
      return {
        lng: parseFloat(values[0]),
        lat: parseFloat(values[1])
      };
    }
    catch (e) {
      return null;
    }
  }
}
```

## layout
form有个全局的layout，参见[form](https://ant.design/components/form/#API)的layout参数。`'horizontal'|'vertical'|'inline'`

若需要嵌套（例如每个input一行，某些行有多个input)，则需要借助[Col](https://ant.design/components/grid/#Col)来控制
``` js
<FormItem
  label="位置"
  required
  {...formItemLayout}
>
  <Col span={6}>
    <FormItem>
      {getFieldDecorator('param1', {
        initialValue: state.editType === 'create' ? "" : data.param1,
        rules: [
          {
            required: true,
            min: 1,
            max: 32
          },
        ],
      })(<Input style={{width: '100%'}}/>)}
    </FormItem>
  </Col>
  <Col span={6}>
    <FormItem>
      {getFieldDecorator('param2', {
        initialValue: state.editType === 'create' ? "" : data.param2,
        rules: [
          {
            required: true,
            type: "integer"
          },
        ],
      })(<InputNumber style={{width: '100%'}}/>)}
    </FormItem>
  </Col>
</FormItem>
```

关于required的红点

若某个表单是必选的，左边的label会有个红色的*。但是对于一行有多个表单（只有一个左侧的label）默认情况下，不会有

解决办法是（见上面代码）在最外层的FormItem加上`required`，内层的无需处理
``` js
// 一行多表单
<FormItem
  label="位置"
  required
  {...formItemLayout}
>
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

# [文件上传](https://ant.design/components/upload-cn/)
以下代码配合后端上传接口实现

``` js
import React from 'react'
import {Upload} from 'antd';
import {connect} from 'dva'
import { config } from '../utils'
import {Icon} from 'antd';

const UploadButton = () => (
  <div>
    <Icon type="plus" />
    <div className="ant-upload-text">点击上传</div>
  </div>
);

const handleChange = (count) => (fileList) => {
  // 1. Limit the number of uploaded files
  //    Only to show two recent uploaded files, and old ones will be replaced by the new
  fileList = fileList.slice(-count);

  // 2. filter successfully uploaded files according to response from server
  fileList = fileList.filter((file) => {
    if (file.response) {
      if (file.response.code !== 0) {
        return false
      }
    }
    return true;
  });

  // 3. read from response and show file link
  fileList = fileList.map((file) => {
    if (file.response) {
      // Component will show file.url as link
      file.url = file.response.data.url;
    }
    return file;
  });

  return fileList
};

const handleOneChange = (fileList) => {
  return handleChange(1)(fileList);
};

const normOneFile = (e) => {
  let res = e;
  if (Array.isArray(e)) {
    res = e;
  } else {
    res = e.fileList
  }

  if (res.length < 1) {
    return ""
  } else {
    return res[0].url;
  }
};

const normFile = (e) => {
  let res = e;
  if (Array.isArray(e)) {
    res = e;
  } else {
    res = e.fileList
  }

  if (res.length < 1) {
    return ""
  } else {
    return res.map(function(object, i) {
      return object.url
    });
  }
};

class ImageUploader extends React.Component {
  constructor(props) {
    super(props);

    const value = this.props.value || [];
    this.state = {
      images: value
    };
  }

  componentWillReceiveProps(nextProps) {
    // Should be a controlled component.
    if ('value' in nextProps && !!nextProps.value) {
      const value = nextProps.value;

      // 排除uploading的
      if(!value) return;

      this.setState({
        images: [
          {
            uid: -1,
            name: 'test.png',
            status: 'done',
            url: value
          }
        ]
      });
    }
  }

  render() {
    const handleChangeImp = (info) => {
      const res = handleOneChange(info.fileList)
      this.setState({images: res});

      const onChange = this.props.onChange;
      if (onChange) {
        onChange(res);
      }
    };

    return (<span>
      <Upload name="file" listType="picture-card" 
        fileList={this.state.images} onChange={handleChangeImp} action="/api/v1/upload">
        {
          this.state.images.length >= 1
            ? null
            : <UploadButton/>
        }
      </Upload>
    </span>);
  }
}

class ImageMultiUploader extends React.Component {
  constructor(props) {
    super(props);

    const value = this.props.value || [];
    this.state = {
      images: value
    };
  }

  componentWillReceiveProps(nextProps) {
    // Should be a controlled component.
    if ('value' in nextProps && !!nextProps.value) {
      const value = nextProps.value;

      // 排除uploading的
      const newValue = value.filter(function(item) {
        return (!item);
      });
      if(newValue.length > 0) return

      let image_urls = [];
      value.forEach(function(item, index) {
        image_urls.push({
          uid: -(index + 1),
          name: 'test' + index + '.png',
          status: 'done',
          url: item
        })
      });

      this.setState({images: image_urls});
    }
  }

  render() {
    const handleChangeImp = (count) => (info) => {
      const res = handleChange(count)(info.fileList)

      this.setState({images: res});

      const onChange = this.props.onChange;
      if (onChange) {
        onChange(res);
      }
    };

    return (<span>
      <Upload name="file" listType="picture-card" fileList={this.state.images} 
      onChange={handleChangeImp(this.props.maxImage)} action="/api/v1/upload">
        {
          this.state.images.length >= this.props.maxImage
            ? null
            : <UploadButton/>
        }
      </Upload>
    </span>);
  }
}

module.exports = {
  ImageUploader,
  ImageMultiUploader,
  normOneFile,
  normFile
};
```
``` js
<FormItem
  {...formItemLayout}
  label="照片"
>
  {getFieldDecorator('image_url', {
    initialValue: state.editType === 'create' ? "" : data.image_url,
    getValueFromEvent: normOneFile,
    rules: [
      {
        required: true,
      },
    ],
  })(
    <ImageUploader />
  )}
</FormItem>
```

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

## [DatePicker](https://ant.design/components/date-picker/)

最新的DatePicker输入输出都是一个moment对象，下面的代码做了个处理，输入输出都是如`2017-12-04`的这种格式

``` js
import React from 'react'
import {DatePicker} from 'antd';
import {connect} from 'dva'
import {config} from '../utils'
import {Icon} from 'antd';
import moment from 'moment';
import PropTypes from 'prop-types';
const RangePicker = DatePicker.RangePicker;

class StringDatePicker extends React.Component {
  constructor(props) {
    super(props);

    const value = this.props.value || [];
    this.state = {
      current_value: undefined
    };
  }

  componentWillReceiveProps(nextProps) {
    // Should be a controlled component.
    if ('value' in nextProps) {
      const value = nextProps.value;
      if(value === undefined) return

      if (!value){
        this.setState({current_value: null});
        return;
      }
      const current_value = moment(value, 'YYYY-MM-DD')
      this.setState({current_value});
    }
  }

  render() {
    const handleChange = (value, dateString) => {
      let current_value = null;
      if(!!value){
        current_value = value.format('YYYY-MM-DD');
      }
      this.setState({
        current_value
      })

      const onChange = this.props.onChange;
      if (onChange) {
        onChange(current_value);
      }
    }

    return (
      <span>
      {this.state.current_value !== undefined && <DatePicker
        defaultValue={this.state.current_value}
        onChange={handleChange}
        showToday={true}
      />}
      </span>
    );
  }
}

StringDatePicker.propTypes = {
  value: PropTypes.string
};


class StringRangePicker extends React.Component {
  constructor(props) {
    super(props);

    const value = this.props.value || [];
    this.state = {
      current_value: undefined
    };
  }

  componentWillReceiveProps(nextProps) {
    // Should be a controlled component.
    if ('value' in nextProps) {
      const value = nextProps.value;
      if(value === undefined) return
      if(!!value && value.length > 0){
        const newValue = value.filter(function(item) {
          return (!!item);
        });
        if(newValue.length !== value.length)
          return;
      }


      if (!value){
        this.setState({current_value: []});
        return;
      }
      const current_value = value.map(item=>moment(item, 'YYYY-MM-DD'))
      this.setState({current_value});
    }
  }

  render() {
    const handleChange = (value, dateString) => {
      let current_value = null;
      if(!!value){
        current_value=value.map(item=>item.format('YYYY-MM-DD'))
      }
      this.setState({
        current_value
      })

      const onChange = this.props.onChange;
      if (onChange) {
        onChange(current_value);
      }
    }

    return (
      <span>
      {this.state.current_value !== undefined && <RangePicker
        defaultValue={this.state.current_value}
        onChange={handleChange}
        showToday={true}
      />}
      </span>
    );
  }
}

StringRangePicker.propTypes = {
  value: PropTypes.arrayOf(PropTypes.string)
};

module.exports = {
  StringDatePicker,
  StringRangePicker,
};

```

# 高德地图
``` js
import React from 'react'
import {mapKey} from '../../utils/config'
import { Map, Marker } from 'react-amap';

const AMap = ({lng,lat}) => {
  const  plugins = [
    'MapType',
    'Scale',
    'OverView',
  ];

  return (
    <div style={{width:"100%",minHeight:"400px"}}>
      <Map amapkey={mapKey}
           plugins={plugins}
           zoom={15}
           center={{longitude: lng, latitude: lat}}
      >
        <Marker position={{longitude: lng, latitude: lat}} />
      </Map>
    </div>
  )
};

export default AMap

```

# Layout

一个常见的管理系统后台，通常都包含一些共有的组件，例如左侧[菜单](https://ant.design/components/menu/)入口，上面导航条，底部说明，[面包屑菜单](https://ant.design/components/breadcrumb/)等

antd admin的实现参考<https://github.com/zuiidea/antd-admin/tree/master/src/components/Layout>

antd admin通过route的一个简单设计，是的每个子页面都无需考虑这些在它外围的基础控件

``` js
const Routers = function ({history, app}) {
  const error = dynamic({
    app,
    component: () => import('./routes/error'),
  });
  const routes = [
    {
      path: route.users,
      models: () => [import('./models/user/index')],
      component: () => import('./routes/user/index'),
    },{
      path: route.userLogin,
      component: () => import('./routes/login/'),
    }
  ];

  return (
    <ConnectedRouter history={history}>
      <App>
        <Switch>
          <Route exact path="/" render={() => (<Redirect to="/user"/>)}/>
          {
            routes.map(({path, ...dynamics}, key) => (
              <Route key={key}
                     exact
                     path={path}
                     component={dynamic({
                       app,
                       ...dynamics,
                     })}
              />
            ))
          }
          <Route component={error}/>
        </Switch>
      </App>
    </ConnectedRouter>
  )
};
```

所有的Page都包含在App Page里面

``` js
/* global window */
import React from 'react'
import NProgress from 'nprogress'
import PropTypes from 'prop-types'
import pathToRegexp from 'path-to-regexp'
import { connect } from 'dva'
import { Layout, Loader } from 'components'
import { classnames, config } from 'utils'
import { Helmet } from 'react-helmet'
import { withRouter } from 'dva/router'
import '../themes/index.less'
import './app.less'
import Error from './error'

const { prefix, openPages,name } = config

const { Header, Bread, Footer, Sider, styles } = Layout
let lastHref

const App = ({ children, dispatch, app, loading, location }) => {
  // 顶部进度条
  if (lastHref !== href) {
    NProgress.start()
    if (!loading.global) {
      NProgress.done()
      lastHref = href
    }
  }

  if (openPages && openPages.includes(pathname)) {
	// 对于特殊的页面，比如login，直接返回Page内容，而不在外面包含公用的组件
    return (<div>
      <Loader fullScreen spinning={loading.effects['app/query']} />
      {children}
    </div>)
  }
  return (
    <div>
      {/* 全局的loading效果 */}
      <Loader fullScreen spinning={loading.effects['app/query']} />
      {/* html title、配置等 */}
      <Helmet>
        <title>{ name }</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <link rel="icon" href={logo} type="image/x-icon" />
        {iconFontJS && <script src={iconFontJS} />}
        {iconFontCSS && <link rel="stylesheet" href={iconFontCSS} />}
      </Helmet>
      <div className={classnames(styles.layout, { [styles.fold]: isNavbar ? false : siderFold }, { [styles.withnavbar]: isNavbar })}>
		// 侧边菜单
        {!isNavbar ? <aside className={classnames(styles.sider, [styles.light])}>
          {siderProps.menu.length === 0 ? null : <Sider {...siderProps} />}
        </aside> : ''}
        <div className={styles.main}>
	      // 顶部导航
          <Header {...headerProps} />
	      // 面包屑菜单
          <Bread {...breadProps} />
		  // 每个页面实际的内容
          <div className={styles.container}>
            <div className={styles.content}>
              {hasPermission ? children : <Error />}
            </div>
          </div>
          // 底部说明
          <Footer />
        </div>
      </div>
    </div>
  )
}

export default withRouter(connect(({ app, loading }) => ({ app, loading }))(App))
```

这种设计的好处是可以利用App的model处理一些通用的东西，比如

全局的错误监听(models/app.js)
``` js
setup({dispatch, history}) {
  // Add a response interceptor
  axios.interceptors.response.use(function (response) {
    // Do something with response data
    return response;
  }, function (error) {

  const {response} = error
  let msg;
  let statusCode;
  if (response && response instanceof Object) {
    statusCode = response.status
    if (statusCode === 401) {
      // Do something with response error
      dispatch({
        type: 'unauthorized',
        payload: error,
      })
    }
  }

  return Promise.reject(error);
});
```

添加路径/路由信息，以及处理一些自动的跳转（登录成功后，存在cookie访问login页面等）
``` js
history.listen((location) => {
  dispatch({
    type: 'updateState',
    payload: {
      locationPathname: location.pathname,
      locationQuery: queryString.parse(location.search),
    },
  });

  const match = pathToRegexp(route.userLoginCB).exec(location.pathname)
  if (!match) {
    dispatch({type: 'query'})
  }
});
```

授权检查拦截
``` js
* query({
          payload,
        }, {call, put, select}) {

  const {locationPathname, locationQuery} = yield select(_ => _.app);
  // 如果不是登录页，那么如果存在就不再查询
  const match = pathToRegexp(route.userLogin).exec(locationPathname);
  if (!match) {
    const {user} = yield select(_ => _.app);
    if ("id" in user) {
      return
    }
  }

  const {data} = yield call(query, payload);
  if (data) {
    const menuData = yield call(menusService.services.query);
    let user = data
    let menu = menuData.data;
    let visit = menu.map(item => item.id);
    yield put({
      type: 'updateState',
      payload: {
        user,
        menu,
      },
    });

    if (locationPathname === route.userLogin) {
      if (locationQuery["url"]) {
        window.location.assign(locationQuery["url"]);
      } else {
        yield put(routerRedux.replace({
          pathname: indexPage,
        }))
      }
    }
  } else if (config.openPages && config.openPages.indexOf(locationPathname) < 0) {
    yield put(routerRedux.push({
      pathname: route.userLogin,
      search: queryString.stringify({
        from: locationPathname,
      }),
    }))
  }
},


* unauthorized(action, {put, select}) {
  const {locationPathname} = yield select(_ => _.app)
  if (locationPathname !== route.userLogin) {
    yield put(routerRedux.push({
      pathname: route.userLogin,
      search: queryString.stringify({
        url: window.location.href
      }),
    }))
  }
},
```

当然一些全局的操作，比如logout处理，左侧菜单栏收房，面包屑菜单处理等也可以在这里一并处理好

## 面包屑菜单
左侧导航菜单的内容，从后端拉取，后端可以根据权限返回有限的数据

antd-admin的实现见<https://github.com/zuiidea/antd-admin/blob/master/src/components/Layout/Menu.js>

输入的菜单数据是个数组，在前端组装一个树结构，由于后端通常也是一个配置来写这些菜单，数组结构抽象树结构很难理解，所以可以考虑将输入也接收树结构，利于后端编辑

为了适配面包屑菜单，后端返回的菜单数据，有一种`隐藏`item,这种item不显示在左边菜单，而仅仅用于面包屑菜单匹配

ant-admin的[Bread](https://github.com/zuiidea/antd-admin/blob/master/src/components/Layout/Bread.js)没有支持导航带参数的链接。比如

``` bash
[首页] => [用户列表] => [用户详情] => [用户编辑]
```

一般的做法，用户详情/编辑的地址会有一个user_id，而不是固定的（例如用户列表这种）

基于`面包屑菜单所需要的参数在当前page中都存在`的假设，可以如下处理
``` js
import React from 'react'
import PropTypes from 'prop-types'
import { Breadcrumb, Icon } from 'antd'
import { Link } from 'react-router-dom'
import pathToRegexp from 'path-to-regexp'
import { queryArray } from 'utils'
import styles from './Bread.less'
import {parseUrl,buildUrl} from '../../utils/url'

const Bread = ({ menu, location }) => {
  // 匹配当前路由
  let pathArray = []
  let current;
  let match;
  for (let index in menu) {
    if (menu[index].route) {
      match = pathToRegexp(menu[index].route).exec(location.pathname);
      if(match){
        current = menu[index];
        break
      }
    }
  }

  const getPathArray = (item) => {
    pathArray.unshift(item);
    if (item.pid) {
      getPathArray(queryArray(menu, item.pid, 'id'))
    }
  };

  if (!current) {
    pathArray.push(menu[0] || {
      id: 1,
      icon: 'laptop',
      name: 'Dashboard',
    });
    pathArray.push({
      id: 404,
      name: 'Not Found',
    })
  } else {
    getPathArray(current)
  }

  // 自动填充
  if(current){
    const urlDatas = parseUrl(location.pathname, current.route);
    pathArray.forEach(function (item) {
      if(item === current || !item.route) return;
      item.route2 = buildUrl(item.route,urlDatas)
    })
  }

  // 递归查找父级
  const breads = pathArray.map((item, key) => {
    const content = (
      <span>{item.icon
        ? <Icon type={item.icon} style={{ marginRight: 4 }} />
        : ''}{item.name}</span>
    )
    return (
      <Breadcrumb.Item key={key}>
        {((pathArray.length - 1) !== key && "route" in item)
          ? <Link to={item.route2 || '#'}>
            {content}
          </Link>
          : content}
      </Breadcrumb.Item>
    )
  })

  return (
    <div className={styles.bread}>
      <Breadcrumb>
        {breads}
      </Breadcrumb>
    </div>
  )
}

Bread.propTypes = {
  menu: PropTypes.array,
  location: PropTypes.object,
}

export default Bread
```

# 省市区

基于动态读取后端数据的省市区控件

数据来源[腾讯地图LBS](https://gitee.com/nxs/nation-province-city-district)

``` js
import React from 'react'
import {Cascader} from 'antd';
import {connect} from 'dva'
import { config } from '../utils'

class Geography extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      options: [],
      current_province: null,
      current_city: null,
      current_area: null,
      default_value: null
    };
  }

  updateConfig(){
    let options = [];
    const {provinces,citys,areas} = this.props
    const state = this.state

    if(!!provinces){
      provinces.forEach(function (item) {
        options.push({
          value: item.id,
          label: item.fullname,
          isLeaf: false,
          type: 'province',
          data: item,
        })
      })
    }

    let currentCitys = null;
    if(!!state.current_province && !!citys) {
      options.find(function(province){
        if(province.value == state.current_province){
          let tmp_citys = []
          citys.forEach(function (item) {
            tmp_citys.push({
              value: item.id,
              label: item.fullname,
              isLeaf: false,
              type: 'city',
              data: item,
            })
          })
          currentCitys = tmp_citys;
          province.children = tmp_citys;
          return true;
        }else{
          return false;
        }
      })
    }

    if(!!currentCitys && !!state.current_city && !!areas){
      currentCitys.find(function(city){
        if(city.value == state.current_city){
          let tmp_areas = []
          areas.forEach(function (item) {
            tmp_areas.push({
              value: item.id,
              label: item.fullname,
              isLeaf: true,
              type: 'area',
              data: item,
            })
          })
          city.children = tmp_areas;
          return true;
        }else{
          return false;
        }
      })
    }

    this.setState({
      options,
    })
  }

  componentWillReceiveProps(nextProps) {
    // Should be a controlled component.
    if (this.state.default_value === null && 'value' in nextProps) {
      let value = nextProps.value
      if(!value){
        value = []
      }

      if(value.length > 0){
        const newValue = value.filter(function(item) {
          return (!!item);
        });
        if(newValue.length !== value.length)
          return;
      }

      this.props.initGeography(value);

      let current_province = null;
      let current_city = null;
      let current_area = null;
      if(nextProps.value.length === 3){
        current_province = nextProps.value[0];
        current_city = nextProps.value[1];
        current_area = nextProps.value[2];
      }

      this.setState({
        default_value: value,
        current_province,
        current_city,
        current_area
      })
    }

    if(this.state.default_value !== null)
      this.updateConfig()
  }

  render() {
    const  loadData = (selectedOptions) => {
      const targetOption = selectedOptions[selectedOptions.length - 1];
      targetOption.loading = true;
      if(targetOption.type == 'province'){
        this.props.getCitys(targetOption.data);
        this.setState({
          current_province: targetOption.data.id,
        })
      }else if(targetOption.type == 'city'){
        this.props.getAreas(targetOption.data);
        this.setState({
          current_city: targetOption.data.id,
        })
      }
      targetOption.loading = false;
    }

    const onChange = (value, selectedOptions) => {
      if(selectedOptions.length == 3 || selectedOptions.length == 0){
        const onChange = this.props.onChange;
        if(selectedOptions.length == 3){
          this.setState({
            current_province: selectedOptions[0].data.id,
            current_city: selectedOptions[1].data.id,
            current_area: selectedOptions[2].data.id,
          })
          if (onChange) {
            onChange(selectedOptions.map(x=>x.data.id));
          }
        }else{
          this.setState({
            current_province: null,
            current_city: null,
            current_area: null,
          })
          if (onChange) {
            onChange([]);
          }
        }
      }
    }

    return (<span>
      {(this.state.options.length > 0 && <Cascader
            defaultValue={this.state.default_value}
            onChange={onChange}
            options={this.state.options}
            loadData={loadData}
            changeOnSelect
          />)}
    </span>);
  }
}


module.exports = {
  Geography
};

```

可复用的model
``` js
import modelExtend from 'dva-model-extend'
import * as share from '../services/share'
import {model} from './common'

const geographyModel = modelExtend(model, {
  state: {
    provinces: [],
    citys: [],
    areas: [],
  },

  effects: {
    * provinces({payload}, {call, put}) {
      const {data} = yield call(share.provinces, {});
      yield put({
        type: 'updateState',
        payload: {
          provinces: data,
        },
      })
    },

    * citys({payload}, {call, put}) {
      const {data} = yield call(share.citys, payload);
      yield put({
        type: 'updateState',
        payload: {
          citys: data,
        },
      })
    },

    * areas({payload}, {call, put}) {
      const {data} = yield call(share.areas, payload);
      yield put({
        type: 'updateState',
        payload: {
          areas: data,
        },
      })
    },

    * selectProvince({payload}, {call, put}) {
      yield put({
        type: 'citys',
        payload: payload['id'],
      });

      yield put({
        type: 'updateState',
        payload: {
          current_province: payload,
        },
      });
    },

    * selectCity({payload}, {call, put}) {
      yield put({
        type: 'areas',
        payload: payload['id'],
      });

      yield put({
        type: 'updateState',
        payload: {
          current_city: payload,
        },
      });
    },

    * initGeography({payload}, {call, put}) {
      let res = yield call(share.provinces, {});
      const provinces = res.data

      // 若没有默认值，只get省份列表
      if(!payload || payload.length != 3){
        yield put({
          type: 'updateState',
          payload: {
            provinces,
            citys: [],
            areas: [],
          },
        })
        return
      }

      const province_id = parseInt(payload[0]);
      const city_id = parseInt(payload[1])
      const area_id = parseInt(payload[2])

      // 默认省份是否匹配
      let current_province = null;
      provinces.find(function(province){
        if(province.id == province_id){
          current_province = province;
          return true
        }else{
          return false
        }
      })
      if(!current_province) return

      // 找到匹配的省份，get 市区
      res = yield call(share.citys, current_province.id);
      const citys = res.data

      let current_city = null;
      citys.find(function(city){
        if(city.id == city_id){
          current_city = city;
          return true
        }else{
          return false
        }
      })
      if(!current_city) return

      res = yield call(share.areas, current_city.id);
      const areas = res.data

      let current_area = null;
      areas.find(function(area){
        if(area.id == area_id){
          current_area = area;
          return true
        }else{
          return false
        }
      })
      if(!current_area) return

      yield put({
        type: 'updateState',
        payload: {
          provinces,
          citys,
          areas,
        },
      })
    },
  },

  reducers: {
    clearGeography(state,{ payload }) {
      return {
        ...state,
        provinces: [],
        citys: [],
        areas: [],
      }
    },
  },

});

module.exports = {
  geographyModel,
};

```

使用

``` js
const geographyProps = {
  provinces: state.provinces,
  citys: state.citys,
  areas: state.areas,
  initGeography(item) {
    dispatch({
      type: `${pageKey}/initGeography`,
      payload: item
    })
  },
  getCitys(item) {
    dispatch({
      type: `${pageKey}/selectProvince`,
      payload: item
    })
  },
  getAreas (item) {
    dispatch({
      type: `${pageKey}/selectCity`,
      payload: item
    })
  },
};
```
``` js
export default modelExtend(geographyModel, {
```
``` js
<FormItem
  label="位置"
  required
  {...formItemLayout}
>
  {getFieldDecorator('geography', {
    initialValue: state.editType === 'create' ? "" : [data.province_code,data.city_code,data.area_code],
    rules: [
      {
        required: true,
      }],
  })(
    <Geography {...geographyProps} />
  )}
</FormItem>
```
