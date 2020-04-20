## 背景

vConsole 是微信团队开源的一个前端开发工具库，方便开发人员测试，调试。与此同时，这篇文章将从源码的角度去更加深入的去了解这个工具的实现过程、架构设计等，以及带给我们的一些思考。

## 说明

文章的代码是从 vConsoleg github 上拉下来的代码，分支为 v3.3.4

## 源码目录设计

vConsole 的源码都在 src 目录下，其目录结构如下。

```
    src
    ├─ core     #核心代码
    ├─ element  #element模块插件
    ├─ log      #log模块插件
    ├─ network  #network模块插件
    ├─ storage  #storage模块插件
    ├─ lib      #工具代码
```

#### core

core 目录包含了 vConsole 核心代码，包括 vConsole 实例化，内置 Plugin 初始化等等。

#### element

element 目录包含了 vConsole 对整个网页的 element 树解析与构建代码。

#### log

log 目录包含了 vConsole 抓取用户输出日志代码，包括对 window.console 劫持等。

#### network

network 目录包含了 vConsole 抓取网络包的实现代码，包括对 window.XMLHttpRequest 部分劫持等等。

#### storage

storage 目录包含了 vCosnole 观察存储状态插件的实现代码，包括观察 Cookies、LocalStorage、SessionStorage 三个存储状态的实现。

#### lib

lib 目录包含了 vConsole 的工具代码、基础代码，包括轻量的 mito 模版解析器、基础插件对象类、全局 API 封装、工具函数等等。

## 源码构建

vConsole 源码是基于 Webpack 构建的，它的构建相关配置都在根目录下。

### 构建脚本

通常一个基于 NPM 托管的项目都会有一个 package.json 文件，它是对项目的描述文件，它的内容实际上是一个标准的 JSON 对象。

我们通常会配置 script 字段作为 NPM 的执行脚本，vConsole 源码构建的脚本如下：

```json
{
  "scripts": {
    "test": "mocha",
    "build": "webpack"
  }
}
```

这里总共有 2 条命令，作用都是构建 vConsole.js，第一条是执行单元测试，第二条是执行打包。

当在命令行运行 npm run build 的时候，实际上就会执行 webpack --config webpack.config.js，接下来我们来看看它实际是怎么构建的。

### 构建过程

我们对于构建过程分析是基于源码的，先打开构建的入口 JS 文件，在 webpack.config.js 中：

```javascript
const pkg = require("./package.json");
const Webpack = require("webpack");
const Path = require("path");
const CopyWebpackPlugin = require("copy-webpack-plugin");

module.exports = {
  mode: "production",
  devtool: false,
  entry: {
    vconsole: Path.resolve(__dirname, "./src/vconsole.js"),
  },
  output: {
    path: Path.resolve(__dirname, "./dist"),
    filename: "[name].min.js",
    library: "VConsole",
    libraryTarget: "umd",
    umdNamedDefine: true,
  },
  module: {
    rules: [
      {
        test: /\.html$/,
        loader: "html-loader?minimize=false",
      },
      {
        test: /\.js$/,
        loader: "babel-loader",
      },
      {
        test: /\.less$/,
        loader: "style-loader!css-loader!less-loader",
      },
    ],
  },
  stats: {
    colors: true,
  },
  plugins: [
    new Webpack.BannerPlugin(
      [
        "vConsole v" + pkg.version + " (" + pkg.homepage + ")",
        "",
        "Tencent is pleased to support the open source community by making vConsole available.",
        "Copyright (C) 2017 THL A29 Limited, a Tencent company. All rights reserved.",
        'Licensed under the MIT License (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at',
        "http://opensource.org/licenses/MIT",
        'Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.',
      ].join("\n")
    ),
    new CopyWebpackPlugin([
      {
        from: Path.resolve(__dirname, "./src/vconsole.d.ts"),
        to: Path.resolve(__dirname, "./dist/vconsole.min.d.ts"),
      },
    ]),
  ],
};
```

这段代码逻辑非常简单，主要是将源码按照遵循 [UMD](https://github.com/umdjs/umd) 规范进行打包，并且在打包后的文件上注入一段声明，同时将 [Typescript](https://www.typescriptlang.org/) 的类型声明复制到打包后的目录下

## 从入口开始

```javascript
// src/vconsole.js
/**
 * A Front-End Console Panel for Mobile Webpage
 */

// global
import "./lib/symbol.js";

// classes
import VConsole from "./core/core.js";

// export
export default VConsole;
```

入口很简单，首先执行<font style="background:#f3f3f3" color=red> import './lib/symbol.js' </font> 进行对 window.Symbol 的兼容性适配以及对 Array 类型对象定义了迭代器以被 for...of 循环使用,最后就是输出 VCosnole 对象，之后我们会对 Core 进行分析。

##### symbol.js

```javascript
// src/lib/symbol.js
if (typeof Symbol === "undefined") {
  window.Symbol = function Symbol() {};

  const key = "__symbol_iterator_key";
  window.Symbol.iterator = key;

  Array.prototype[key] = function symbolIterator() {
    const that = this;
    let i = 0;
    return {
      next() {
        return {
          done: that.length === i,
          value: that.length === i ? undefined : that[i++],
        };
      },
    };
  };
}
```

## Core 分析

当我们使用 vConsole 的时候,我们首先都是通过 new VConsole() 的形式进行调用，而此时执行的就是 Core 内部的构建函数，代码如下

```javascript
// src\core\core.js
class VConsole {
  constructor(opt) {
    // 过程1 采用单例的设计模式先判断vConsole这个工具是否初始化且渲染到dom结点上
    if (!!$.one(VCONSOLE_ID)) {
      console.debug("vConsole is already exists.");
      return;
    }
    let that = this;

    this.version = pkg.version;
    this.$dom = null;

    this.isInited = false;
    this.option = {
      defaultPlugins: ["system", "network", "element", "storage"],
    };

    this.activedTab = "";
    this.tabList = [];
    this.pluginList = {};

    this.switchPos = {
      x: 10, // right
      y: 10, // bottom
      startX: 0,
      startY: 0,
      endX: 0,
      endY: 0,
    };

    // export helper functions to public
    this.tool = tool;
    this.$ = $;

    // merge options
    if (tool.isObject(opt)) {
      for (let key in opt) {
        this.option[key] = opt[key];
      }
    }

    // add built-in plugins
    this._addBuiltInPlugins();

    // try to init
    let _onload = function () {
      if (that.isInited) {
        return;
      }
      that._render();
      that._mockTap();
      that._bindEvent();
      that._autoRun();
    };
    if (document !== undefined) {
      if (document.readyState === "loading") {
        $.bind(window, "DOMContentLoaded", _onload);
      } else {
        _onload();
      }
    } else {
      // if document does not exist, wait for it
      let _timer;
      let _pollingDocument = function () {
        if (!!document && document.readyState == "complete") {
          _timer && clearTimeout(_timer);
          _onload();
        } else {
          _timer = setTimeout(_pollingDocument, 1);
        }
      };
      _timer = setTimeout(_pollingDocument, 1);
    }
  }

  //...
}
```

首先这段构建函数里主要分为一下几个过程

1.  采用单例的设计模式先判断 vConsole 这个工具是否初始化且渲染到 dom 结点上
    ```javascript
    if (!!$.one(VCONSOLE_ID)) {
      console.debug("vConsole is already exists.");
      return;
    }
    ```
2.  内部 options 与 外部传进来的 options 进行合并，以及内部的操作 element 的方法、工具方法绑定在 Core 对象当中

    ```javascript
    let that = this;

    this.version = pkg.version;
    this.$dom = null;

    this.isInited = false;
    this.option = {
      defaultPlugins: ["system", "network", "element", "storage"],
    };

    this.activedTab = "";
    this.tabList = [];
    this.pluginList = {};

    this.switchPos = {
      x: 10, // right
      y: 10, // bottom
      startX: 0,
      startY: 0,
      endX: 0,
      endY: 0,
    };

    // export helper functions to public
    this.tool = tool;
    this.$ = $;

    // merge options
    if (tool.isObject(opt)) {
      for (let key in opt) {
        this.option[key] = opt[key];
      }
    }
    ```

3.  根据 options 实例化插件,如默认情况下的 element，log，network，storage 插件, 在这个过程有个小细节要注意的是这里仅仅包含了 plugin 对象的实例化的这一过程，**_但是不一定包含插件 ui 插入到 dom 结点上（这个过程以及原因我们接下来会进行分析）_**


    ```javascript
    // add built-in plugins
    this._addBuiltInPlugins();
    ```

4.  最后就是一个当 document 整个文档载入完成后进行一个完整的实例到 ui 渲染，ui 事件绑定，执行插件生命周期的一个过程。调用 that.\_render 是将整个 vConsole 的 ui 组件插入到 document 文档中,\_mockTap 与\_bindEvent 就是绑定 ui 的相关事件，\_autoRun 就是调用插件内部的生命周期。
    ```javascript
    // ...
    // try to init
    let _onload = function () {
      if (that.isInited) {
        return;
      }
      that._render();
      that._mockTap();
      that._bindEvent();
      that._autoRun();
    };
    // ...
    ```

#### \_addBuiltInPlugins 过程

首先我们先了解一个基础的插件包含了什么？代码如下：

```javascript
// src\lib\plugin.js
class VConsolePlugin {
  constructor(id, name = "newPlugin") {
    this.id = id;
    this.name = name;
    this.isReady = false;

    this.eventList = {};
  }

  get id() {
    return this._id;
  }

  set id(value) {
    if (!value) {
      throw "Plugin ID cannot be empty";
    }
    this._id = value.toLowerCase();
  }

  get name() {
    return this._name;
  }

  set name(value) {
    if (!value) {
      throw "Plugin name cannot be empty";
    }
    this._name = value;
  }

  get vConsole() {
    return this._vConsole || undefined;
  }

  set vConsole(value) {
    if (!value) {
      throw "vConsole cannot be empty";
    }
    this._vConsole = value;
  }

  /**
   * register an event
   * @public
   * @param string
   * @param function
   */
  on(eventName, callback) {
    this.eventList[eventName] = callback;
    return this;
  }

  /**
   * trigger an event
   * @public
   * @param string
   * @param mixed
   */
  trigger(eventName, data) {
    if (typeof this.eventList[eventName] === "function") {
      // registered by `.on()` method
      this.eventList[eventName].call(this, data);
    } else {
      // registered by `.onXxx()` method
      let method =
        "on" + eventName.charAt(0).toUpperCase() + eventName.slice(1);
      if (typeof this[method] === "function") {
        this[method].call(this, data);
      }
    }
    return this;
  }
}
```

一个插件进行抽象化后包含了插件 id，插件名 name，状态值 isReady，订阅中心 eventlist，发布者 trigger，实例化 core 之后的\_vConsole 对象,以及一些对属性劫持的方法等

其次，任何一个插件都要继承这个基础的抽象插件或者包含基础插件要求的属性来实现各自业务逻辑，他们的表现形式如 storage 插件：

```javascript
  //src\storage\storage.js
  class VConsoleStorageTab extends VConsolePlugin {

        constructor(...args) {
          super(...args);

          this.$tabbox = $.render(tplTabbox, {});
          this.currentType = ''; // cookies, localstorage, ...
          this.typeNameMap = {
            'cookies': 'Cookies',
            'localstorage': 'LocalStorage',
            'sessionstorage': 'SessionStorage'
          }
      //...
  }
```

最后回到 Core 文件的源码分析上，

```javascript
  // src\core\core.js
  //...

  _addBuiltInPlugins() {
    // add default log plugin
    this.addPlugin(new VConsoleDefaultPlugin('default', 'Log'));

    // add other built-in plugins according to user's config
    const list = this.option.defaultPlugins;
    const plugins = {
      'system': { proto: VConsoleSystemPlugin, name: 'System' },
      'network': { proto: VConsoleNetworkPlugin, name: 'Network' },
      'element': { proto: VConsoleElementPlugin, name: 'Element' },
      'storage': { proto: VConsoleStoragePlugin, name: 'Storage' }
    };
    if (!!list && tool.isArray(list)) {
      for (let i = 0; i < list.length; i++) {
        let tab = plugins[list[i]];
        if (!!tab) {
          this.addPlugin(new tab.proto(list[i], tab.name));
        } else {
          console.debug('Unrecognized default plugin ID:', list[i]);
        }
      }
    }
  }

  //...

    /**
   * add a new plugin
   * @public
   * @param object VConsolePlugin object
   * @return boolean
   */
  addPlugin(plugin) {
    // ignore this plugin if it has already been installed

    if (this.pluginList[plugin.id] !== undefined) {
      console.debug('Plugin ' + plugin.id + ' has already been added.');
      return false;
    }
    this.pluginList[plugin.id] = plugin;
    // init plugin only if vConsole is ready
    if (this.isInited) {
      this._initPlugin(plugin);
      // if it's the first plugin, show it by default
      if (this.tabList.length == 1) {
        this.showTab(this.tabList[0]);
      }
    }
    return true;
  }
  //...
```

在 new VConsole()过程中中主要执行一个逻辑即维护一个插件 key-value 对象，用于每个对象值允许初始化一次，但是这个过程中是不会执行\_initPlugin 的,因为此时此刻 vConsole 还没初始化完成，而这个标志的更改是在 doucument 加载完成后，执行\_autoRun 后的一个事件中完成的,同时因为 vConsole 的 ui 是在\_render 中完成的，而且这个插件的 ui 的父节点就是 vConsole 的 ui，如果此时进行初始化会报出异常错误。

#### \_render 过程

```javascript
// src\core\core.js
  _render() {
    if (!$.one(VCONSOLE_ID)) {
      let e = document.createElement('div');
      e.innerHTML = tpl;
      document.documentElement.insertAdjacentElement('beforeend', e.children[0]);
    }
    this.$dom = $.one(VCONSOLE_ID);

    // reposition switch button
    let $switch = $.one('.vc-switch', this.$dom);
    let switchX = tool.getStorage('switch_x') * 1,
      switchY = tool.getStorage('switch_y') * 1;
    if (switchX || switchY) {
      // check edge
      if (switchX + $switch.offsetWidth > document.documentElement.offsetWidth) {
        switchX = document.documentElement.offsetWidth - $switch.offsetWidth;
      }
      if (switchY + $switch.offsetHeight > document.documentElement.offsetHeight) {
        switchY = document.documentElement.offsetHeight - $switch.offsetHeight;
      }
      if (switchX < 0) { switchX = 0; }
      if (switchY < 0) { switchY = 0; }
      this.switchPos.x = switchX;
      this.switchPos.y = switchY;
      $.one('.vc-switch').style.right = switchX + 'px';
      $.one('.vc-switch').style.bottom = switchY + 'px';
    }

    // modify font-size
    let dpr = window.devicePixelRatio || 1;
    let viewportEl = document.querySelector('[name="viewport"]');
    if (viewportEl && viewportEl.content) {
      let initialScale = viewportEl.content.match(/initial\-scale\=\d+(\.\d+)?/);
      let scale = initialScale ? parseFloat(initialScale[0].split('=')[1]) : 1;
      if (scale < 1) {
        this.$dom.style.fontSize = 13 * dpr + 'px';
      }
    }

    // remove from less to present transition effect
    $.one('.vc-mask', this.$dom).style.display = 'none';
  };
```

render 的过程就是就是一个将一个 vConsole 的 ui 模版[tpl](https://github.com/Tencent/vConsole/blob/dev/doc/helper_functions.md)插入到 document 文档中

#### \_autoRun 过程

当执行到这个过程的时候,对于插件的容器 vConsole 来讲，此时此刻已经是初始化完成了，而到这个过程的时候，就是真正的执行插件内部初始化逻辑的,执行完毕后发布一个 ready 的事件

```javascript
  // src\core\core.js

  /**
   * auto run after initialization
   * @private
   */
  _autoRun() {
    this.isInited = true;

    // init plugins
    for (let id in this.pluginList) {
      this._initPlugin(this.pluginList[id]);
    }

    // show first tab
    if (this.tabList.length > 0) {
      this.showTab(this.tabList[0]);
    }

    this.triggerEvent('ready');
  }
```

### 小结

通过源码的分析，我们可以总结出 vConsole 核心的简单的初始化流程如下图

![build](https://github.com/rhymedys/rhy-vconsole-analyse/blob/master/初始化流程.png)

## Plugin 分析

这里我们主要分析下插件初始化过程，主要以 Network 为例，了解整个插件的设计。在上文 Core 分析中，我们得知在\_addBuiltInPlugins 这个过程中会执行一次 Plugin 的实例化过程，在 Network 下代码如下：

```javascript
// src\network\network.js
class VConsoleNetworkTab extends VConsolePlugin {
  // ...
  constructor(...args) {
    super(...args);
    // 容器布局
    this.$tabbox = $.render(tplTabbox, {});
    // header布局
    this.$header = null;
    // 请求列表的key value对象
    this.reqList = {}; // URL as key, request item as value
    // 请求列表ui的key value对象
    this.domList = {}; // URL as key, dom item as value
    // 插件是否就绪？
    this.isReady = false;
    // 插件ui是否显示？
    this.isShow = false;
    // 请求列表ui是否滑动到底部？
    this.isInBottom = true; // whether the panel is in the bottom
    // XMLHttpRequest对象的原始 open 函数
    this._open = undefined; // the origin function
    // XMLHttpRequest对象的原始 send 函数
    this._send = undefined;

    this.mockAjax();
  }
  // ...
}
```

在这段代码,首先会调用 mito.js **_(这是一个内置的轻量的 ui 模版渲染引擎，可以单独抽出来使用)_** 中的 render 方法将插件容器的一段字符串 dom 进行 doucument 树对象结构化并保存为 \$tabbox，即将以下代码

```javascript
const tplTabbox = '<div class="vc-table"><div class="vc-log"></div></div>';
```

转换成

```html
<div class="vc-table">
  <div class="vc-log"></div>
</div>
```

然后进行一些基础配置的初始化，比如插件的就绪状态 isReady，显示状态 isShow 等等的一些初始化配置，再然调用

```javascript
this.mockAjax();
```

```javascript
 mockAjax() {
    let _XMLHttpRequest = window.XMLHttpRequest;
    if (!_XMLHttpRequest) { return; }

    let that = this;
    let _open = window.XMLHttpRequest.prototype.open,
        _send = window.XMLHttpRequest.prototype.send;
    that._open = _open;
    that._send = _send;

    // mock open()
    window.XMLHttpRequest.prototype.open = function() {
      let XMLReq = this;
      let args = [].slice.call(arguments),
          method = args[0],
          url = args[1],
          id = that.getUniqueID();
      let timer = null;

      // may be used by other functions
      XMLReq._requestID = id;
      XMLReq._method = method;
      XMLReq._url = url;

      // mock onreadystatechange
      let _onreadystatechange = XMLReq.onreadystatechange || function() {};
      let onreadystatechange = function() {

        let item = that.reqList[id] || {};

        // update status
        item.readyState = XMLReq.readyState;
        item.status = 0;
        if (XMLReq.readyState > 1) {
          item.status = XMLReq.status;
        }
        item.responseType = XMLReq.responseType;

        if (XMLReq.readyState == 0) {
          // UNSENT
          if (!item.startTime) {
            item.startTime = (+new Date());
          }
        } else if (XMLReq.readyState == 1) {
          // OPENED
          if (!item.startTime) {
            item.startTime = (+new Date());
          }
        } else if (XMLReq.readyState == 2) {
          // HEADERS_RECEIVED
          item.header = {};
          let header = XMLReq.getAllResponseHeaders() || '',
              headerArr = header.split("\n");
          // extract plain text to key-value format
          for (let i=0; i<headerArr.length; i++) {
            let line = headerArr[i];
            if (!line) { continue; }
            let arr = line.split(': ');
            let key = arr[0],
                value = arr.slice(1).join(': ');
            item.header[key] = value;
          }
        } else if (XMLReq.readyState == 3) {
          // LOADING
        } else if (XMLReq.readyState == 4) {
          // DONE
          clearInterval(timer);
          item.endTime = +new Date(),
          item.costTime = item.endTime - (item.startTime || item.endTime);
          item.response = XMLReq.response;
        } else {
          clearInterval(timer);
        }

        if (!XMLReq._noVConsole) {
          that.updateRequest(id, item);
        }
        return _onreadystatechange.apply(XMLReq, arguments);
      };
      XMLReq.onreadystatechange = onreadystatechange;

      // some 3rd libraries will change XHR's default function
      // so we use a timer to avoid lost tracking of readyState
      let preState = -1;
      timer = setInterval(function() {
        if (preState != XMLReq.readyState) {
          preState = XMLReq.readyState;
          onreadystatechange.call(XMLReq);
        }
      }, 10);

      return _open.apply(XMLReq, args);
    };

    // mock send()
    window.XMLHttpRequest.prototype.send = function() {
      let XMLReq = this;
      let args = [].slice.call(arguments),
          data = args[0];

      let item = that.reqList[XMLReq._requestID] || {};
      item.method = XMLReq._method.toUpperCase();

      let query = XMLReq._url.split('?'); // a.php?b=c&d=?e => ['a.php', 'b=c&d=', '?e']
      item.url = query.shift(); // => ['b=c&d=', '?e']

      if (query.length > 0) {
        item.getData = {};
        query = query.join('?'); // => 'b=c&d=?e'
        query = query.split('&'); // => ['b=c', 'd=?e']
        for (let q of query) {
          q = q.split('=');
          item.getData[ q[0] ] = decodeURIComponent(q[1]);
        }
      }

      if (item.method == 'POST') {

        // save POST data
        if (tool.isString(data)) {
          let arr = data.split('&');
          item.postData = {};
          for (let q of arr) {
            q = q.split('=');
            item.postData[ q[0] ] = q[1];
          }
        } else if (tool.isPlainObject(data)) {
          item.postData = data;
        }

      }

      if (!XMLReq._noVConsole) {
        that.updateRequest(XMLReq._requestID, item);
      }

      return _send.apply(XMLReq, args);
    };

  };
```

对 XMLHttpRequest 的对象中的原始 open 跟 send 方法进行劫持，以达到能够监听我们使用 XMLHttpRequest 对象发起的网络请求，**_在这里，我们发现了几个问题_**

1.  如果我们发起网络请求的对象是借由 Fetch 对象发起的，那么此时此刻这个插件是不能监听到我们发起的网络请求，也就是 Element 插件暂时还没对 Fetch 对象发起的网络请求进行适配
2.  在使用过程中我们发现了，这个工具不能够抓取到我们发起的 XMLHttpRequest 网络请求的 request headers 这个对象，我们思考下其实这个工具能够劫持 open 跟 send 函数来获取我们发送过去的 query、json 等数据，因此可以稍微改造一下 mockAjax 的实现，即

    ```javascript
    mockAjax() {
      // ...

      let _open = window.XMLHttpRequest.prototype.open,
          _send = window.XMLHttpRequest.prototype.send,
          _setRequestHeader = window.XMLHttpRequest.prototype.setRequestHeader;
      that._open = _open;
      that._send = _send;
      that._setRequestHeader = _setRequestHeader;

      window.XMLHttpRequest.prototype.setRequestHeader = function () {
        let args = [].slice.call(arguments);
        // 实现劫持逻辑
        // ...
        _setRequestHeader.apply(this, args);
      };

      // ...
    }
    ```

    这样子虽然不能劫持到所有的请求，但是在我平时前端上层业务上关注的较多的就是上层业务通过 setRequestHeader 来设置请求头的 header，当然如果是我们自己的应用（android，electron 等）的 Webview 我们可以通过改造内核的方式去实现请求头的完整显示。

在 Core 中调用 \_addBuiltInPlugins 进行 Plugin 的实例化后，我们接下来分析调用\_initPlugin 的逻辑，

```javascript
  _initPlugin(plugin) {
    let that = this;
    plugin.vConsole = this;
    // start init
    plugin.trigger('init');
    // render tab (if it is a tab plugin then it should has tab-related events)
    plugin.trigger('renderTab', function (tabboxHTML) {
      // add to tabList
      that.tabList.push(plugin.id);
      // render tabbar
      let $tabbar = $.render(tplTabbar, {
        id: plugin.id,
        name: plugin.name
      });
      $.one('.vc-tabbar', that.$dom).insertAdjacentElement('beforeend', $tabbar);
      // render tabbox
      let $tabbox = $.render(tplTabbox, {
        id: plugin.id
      });
      if (!!tabboxHTML) {
        if (tool.isString(tabboxHTML)) {
          $tabbox.innerHTML += tabboxHTML;
        } else if (tool.isFunction(tabboxHTML.appendTo)) {
          tabboxHTML.appendTo($tabbox);
        } else if (tool.isElement(tabboxHTML)) {
          $tabbox.insertAdjacentElement('beforeend', tabboxHTML);
        }
      }
      $.one('.vc-content', that.$dom).insertAdjacentElement('beforeend', $tabbox);
    });
    // render top bar
    plugin.trigger('addTopBar', function (btnList) {
      if (!btnList) {
        return;
      }
      let $topbar = $.one('.vc-topbar', that.$dom);
      for (let i = 0; i < btnList.length; i++) {
        let item = btnList[i];
        let $item = $.render(tplTopBarItem, {
          name: item.name || 'Undefined',
          className: item.className || '',
          pluginID: plugin.id
        });
        if (item.data) {
          for (let k in item.data) {
            $item.dataset[k] = item.data[k];
          }
        }
        if (tool.isFunction(item.onClick)) {
          $.bind($item, 'click', function (e) {
            let enable = item.onClick.call($item);
            if (enable === false) {
              // do nothing
            } else {
              $.removeClass($.all('.vc-topbar-' + plugin.id), 'vc-actived');
              $.addClass($item, 'vc-actived');
            }
          });
        }
        $topbar.insertAdjacentElement('beforeend', $item);
      }
    });
    // render tool bar
    plugin.trigger('addTool', function (toolList) {
      if (!toolList) {
        return;
      }
      let $defaultBtn = $.one('.vc-tool-last', that.$dom);
      for (let i = 0; i < toolList.length; i++) {
        let item = toolList[i];
        let $item = $.render(tplToolItem, {
          name: item.name || 'Undefined',
          pluginID: plugin.id
        });
        if (item.global == true) {
          $.addClass($item, 'vc-global-tool');
        }
        if (tool.isFunction(item.onClick)) {
          $.bind($item, 'click', function (e) {
            item.onClick.call($item);
          });
        }
        $defaultBtn.parentNode.insertBefore($item, $defaultBtn);
      }
    });
    // end init
    plugin.isReady = true;
    plugin.trigger('ready');
  }
```

这段逻辑主要有以下几个步骤：

1.  给每个 Plugin 注入一个实例化后的 VConsole 对象
2.  执行 init，renderTab， addTopBar，addTool，ready 的一个生命周期，主要是一些 Plugin 内部的 ui 插入到 vConsole 这个父节点的过程以及初始化完成后的逻辑

### 小结

通过以上的分析，我们得知一个 Plugin 的完整初始化流程如下图：

![plugininit](https://github.com/rhymedys/rhy-vconsole-analyse/blob/master/plugininit.png)

其他的 element、log、storage 等插件都遵循以上的分析，各自业务需自行查看源码

## 总结

通过以上分析我们可知整个工具架构是这样的：

![build](https://github.com/rhymedys/rhy-vconsole-analyse/blob/master/build.png)

lib 层：提供引擎，工具给上层的 plugin 与应用层使用，同时定义了基础插件抽象类给 plugin 继承使用

plugin 层：继承基础插件抽象类对象实现插件的业务化，向上层提供具体的功能

core 层：集成插件形成应用，统一管理，调度插件的实例化初始化过程，最终提供一个可用的应用


## 参考

[vConsole](https://github.com/Tencent/vConsole)
