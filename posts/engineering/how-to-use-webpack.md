---
title: "Webpack + React + ES6 技术栈全套配置指南"
date: 2016-07-14
draft: false
categories: []
tags: [webpack]
---

因为最近在工作中尝试了 [webpack](https://github.com/webpack/webpack)、[react](https://github.com/facebook/react)、[redux](https://github.com/reactjs/redux)、[es6](http://babeljs.io/docs/learn-es2015/) 技术栈，所以总结出了一套 [boilerplate](https://github.com/xiaoyann/webpack-react-redux-es6-boilerplate)，以便下次做项目时可以快速开始，并进行持续优化。对应的项目地址：[webpack-react-redux-es6-boilerplate](https://github.com/xiaoyann/webpack-react-redux-es6-boilerplate)

该项目的 webpack 配置做了不少优化，所以构建速度还不错。文章的最后还对使用 webpack 的问题及性能优化作出了总结。

## 项目结构规划

每个模块相关的 css、img、js 文件都放在一起，比较直观，删除模块时也会方便许多。测试文件也同样放在一起，哪些模块有没有写测试，哪些测试应该一起随模块删除，一目了然。

```shell
build
|-- webpack.config.js               # 公共配置
|-- webpack.dev.js                  # 开发配置
|-- webpack.release.js              # 发布配置
docs                                # 项目文档
node_modules                        
src                                 # 项目源码
|-- conf                            # 配置文件
|-- pages                           # 页面目录
|   |-- page1                       
|   |   |-- index.js                # 页面逻辑
|   |   |-- index.scss              # 页面样式
|   |   |-- img                     # 页面图片
|   |   |   |-- xx.png      
|   |   |-- __tests__               # 测试文件
|   |   |   |-- xx.js
|   |-- app.html                    # 入口页
|   |-- app.js                      # 入口JS
|-- components                      # 组件目录
|   |-- loading
|   |   |-- index.js
|   |   |-- index.scss
|   |   |-- __tests__               
|   |   |   |-- xx.js
|-- js
|   |-- actions
|   |   |-- index.js
|   |   |-- __tests__               
|   |   |   |-- xx.js
|   |-- reducers 
|   |   |-- index.js
|   |   |-- __tests__               
|   |   |   |-- xx.js
|   |-- xx.js         
|-- css                             # 公共CSS目录
|   |-- common.scss
|-- img                             # 公共图片目录
|   |-- xx.png
tests                               # 其他测试文件
package.json                        
READNE.md                           
```

## 要完成的功能

1. 编译 jsx、es6、scss 等资源
2. 自动引入静态资源到相应 html 页面
3. 实时编译和刷新浏览器
4. 按指定模块化规范自动包装模块
5. 自动给 css 添加浏览器内核前缀
6. 按需打包合并 js、css
7. 压缩 js、css、html
8. 图片路径处理、压缩、CssSprite
9. 对文件使用 hash 命名，做强缓存
10. 语法检查
11. 全局替换指定字符串
12. 本地接口模拟服务
13. 发布到远端机

针对以上的几点功能，接下来将一步一步的来完成这个 [boilerplate](https://github.com/xiaoyann/webpack-react-redux-es6-boilerplate) 项目， 并记录下每一步的要点。


### 准备工作

1、根据前面的项目结构规划创建项目骨架

```shell
$ make dir webpack-react-redux-es6-boilerplate
$ cd webpack-react-redux-es6-boilerplate
$ mkdir build docs src mock tests
$ touch build/webpack.config.js build/webpack.dev.js build/webpack.release.js
// 创建 package.json
$ npm init
$ ...
```

2、安装最基本的几个 npm 包

```shell
$ npm i webpack webpack-dev-server --save-dev
$ npm i react react-dom react-router redux react-redux redux-thunk --save
```

3、编写示例代码，最终代码直接查看 [boilerplate](https://github.com/xiaoyann/webpack-react-redux-es6-boilerplate/tree/master/src/pages)

4、根据 [webpack](http://webpack.github.io/docs/) 文档编写最基本的 webpack 配置，直接使用 NODE API 的方式

```js
/* webpack.config.js */

var webpack = require('webpack');

// 辅助函数
var utils = require('./utils');
var fullPath  = utils.fullPath;
var pickFiles = utils.pickFiles;

// 项目根路径
var ROOT_PATH = fullPath('../');
// 项目源码路径
var SRC_PATH = ROOT_PATH + '/src';
// 产出路径
var DIST_PATH = ROOT_PATH + '/dist';

// 是否是开发环境
var __DEV__ = process.env.NODE_ENV !== 'production';

// conf
var alias = pickFiles({
  id: /(conf\/[^\/]+).js$/,
  pattern: SRC_PATH + '/conf/*.js'
});

// components
alias = Object.assign(alias, pickFiles({
  id: /(components\/[^\/]+)/,
  pattern: SRC_PATH + '/components/*/index.js'
}));

// reducers
alias = Object.assign(alias, pickFiles({
  id: /(reducers\/[^\/]+).js/,
  pattern: SRC_PATH + '/js/reducers/*'
}));

// actions
alias = Object.assign(alias, pickFiles({
  id: /(actions\/[^\/]+).js/,
  pattern: SRC_PATH + '/js/actions/*'
}));


var config = {
  context: SRC_PATH,
  entry: {
    app: ['./pages/app.js']
  },
  output: {
    path: DIST_PATH,
    filename: 'js/bundle.js'
  },
  module: {},
  resolve: {
    alias: alias
  },
  plugins: [
    new webpack.DefinePlugin({
      // http://stackoverflow.com/questions/30030031/passing-environment-dependent-variables-in-webpack
      "process.env.NODE_ENV": JSON.stringify(process.env.NODE_ENV || 'development')
    })
  ]
};

module.exports = config;
```

```js
/* webpack.dev.js */

var webpack = require('webpack');
var WebpackDevServer = require('webpack-dev-server');
var config = require('./webpack.config');
var utils = require('./utils');

var PORT = 8080;
var HOST = utils.getIP();
var args = process.argv;
var hot = args.indexOf('--hot') > -1;
var deploy = args.indexOf('--deploy') > -1;

// 本地环境静态资源路径
var localPublicPath = 'http://' + HOST + ':' + PORT + '/';

config.output.publicPath = localPublicPath; 
config.entry.app.unshift('webpack-dev-server/client?' + localPublicPath);

new WebpackDevServer(webpack(config), {
  hot: hot,
  inline: true,
  compress: true,
  stats: {
    chunks: false,
    children: false,
    colors: true
  },
  // Set this as true if you want to access dev server from arbitrary url.
  // This is handy if you are using a html5 router.
  historyApiFallback: true,
}).listen(PORT, HOST, function() {
  console.log(localPublicPath);
});
```

上面的配置写好后就可以开始构建了

```shell
$ node build/webpack.dev.js
```

因为项目中使用了 jsx、es6、scss，所以还要添加相应的 loader，否则会报如下类似错误：

```js
ERROR in ./src/pages/app.js
Module parse failed: /Users/xiaoyan/working/webpack-react-redux-es6-boilerplate/src/pages/app.js Unexpected token (18:6)
You may need an appropriate loader to handle this file type.
```

### 编译 jsx、es6、scss 等资源

* 使用 [bael](http://babeljs.io/) 和 [babel-loader](https://github.com/babel/babel-loader) 编译 jsx、es6
* 安装插件: [babel-preset-es2015](http://babeljs.io/docs/plugins/preset-es2015/) 用于解析 `es6`
* 安装插件：[babel-preset-react](http://babeljs.io/docs/plugins/preset-react/) 用于解析 `jsx`

```js
// 首先需要安装 babel 
$ npm i babel-core --save-dev
// 安装插件 
$ npm i babel-preset-es2015 babel-preset-react --save-dev
// 安装 loader
$ npm i babel-loader --save-dev
```

在项目根目录创建 `.babelrc` 文件:

```
{
  "presets": ["es2015", "react"]
}
```

在 webpack.config.js 里添加：
```js
// 使用缓存
var CACHE_PATH = ROOT_PATH + '/cache';
// loaders
config.module.loaders = [];
// 使用 babel 编译 jsx、es6
config.module.loaders.push({
  test: /\.js$/,
  exclude: /node_modules/,
  include: SRC_PATH,
  // 这里使用 loaders ，因为后面还需要添加 loader
  loaders: ['babel?cacheDirectory=' + CACHE_PATH]
});

```

接下来使用 [sass-loader](https://github.com/jtangelder/sass-loader) 编译 sass:

```shell
$ npm i sass-loader node-sass css-loader style-loader --save-dev
```

* [css-loader](https://github.com/webpack/css-loader) 用于将 css 当做模块一样来 `import`
* [style-loader](https://github.com/webpack/style-loader) 用于自动将 css 添加到页面


在 webpack.config.js 里添加：

```js
// 编译 sass
config.module.loaders.push({
  test: /\.(scss|css)$/,
  loaders: ['style', 'css', 'sass']
});
```

### 自动引入静态资源到相应 html 页面

* 使用 [html-webpack-plugin](https://github.com/ampedandwired/html-webpack-plugin)

```js
$ npm i html-webpack-plugin --save-dev
```

在 webpack.config.js 里添加：
```js
// html 页面
var HtmlwebpackPlugin = require('html-webpack-plugin');
config.plugins.push(
  new HtmlwebpackPlugin({
    filename: 'index.html',
    chunks: ['app'],
    template: SRC_PATH + '/pages/app.html'
  })
);
```

至此，整个项目就可以正常跑起来了

```shell
$ node build/webpack.dev.js
```

### 实时编译和刷新浏览器

完成前面的配置后，项目就已经可以实时编译和自动刷新浏览器了。接下来就配置下热更新，使用 [react-hot-loader](https://github.com/gaearon/react-hot-loader)：

```js
$ npm i react-hot-loader --save-dev
```

因为热更新只需要在开发时使用，所以在 webpack.dev.config 里添加如下代码：

```js
// 开启热替换相关设置
if (hot === true) {
  config.entry.app.unshift('webpack/hot/only-dev-server');
  // 注意这里 loaders[0] 是处理 .js 文件的 loader
  config.module.loaders[0].loaders.unshift('react-hot');
  config.plugins.push(new webpack.HotModuleReplacementPlugin());
}

```
执行下面的命令，并尝试更改 js、css：
```shell
$ node build/webpack.dev.js --hot
```

### 按指定模块化规范自动包装模块

webpack 支持 CommonJS、AMD 规范，具体如何使用直接查看文档

### 自动给 css 添加浏览器内核前缀 

使用 [postcss-loader](https://github.com/postcss/postcss-loader)

```shell
npm i postcss-loader precss autoprefixer --save-dev
```

在 webpack.config.js 里添加：

```js
// 编译 sass
config.module.loaders.push({
  test: /\.(scss|css)$/,
  loaders: ['style', 'css', 'sass', 'postcss']
});

// css autoprefix
var precss = require('precss');
var autoprefixer = require('autoprefixer');
config.postcss = function() {
  return [precss, autoprefixer];
}
```

### 打包合并 js、css

webpack 默认将所有模块都打包成一个 bundle，并提供了 [Code Splitting](http://webpack.github.io/docs/code-splitting.html) 功能便于我们按需拆分。在这个例子里我们把框架和库都拆分出来：

在 webpack.config.js 添加：
```js
config.entry.lib = [
  'react', 'react-dom', 'react-router',
  'redux', 'react-redux', 'redux-thunk'
]

config.output.filename = 'js/[name].js';

config.plugins.push(
    new webpack.optimize.CommonsChunkPlugin('lib', 'js/lib.js')
);

// 别忘了将 lib 添加到 html 页面
// chunks: ['app', 'lib']
```

如何拆分 CSS：[separate css bundle](http://webpack.github.io/docs/stylesheets.html)

### 压缩 js、css、html、png 图片

压缩资源最好只在生产环境时使用

```js
// 压缩 js、css
config.plugins.push(
    new webpack.optimize.UglifyJsPlugin({
        compress: {
            warnings: false
        }
    })
);

// 压缩 html
// html 页面
var HtmlwebpackPlugin = require('html-webpack-plugin');
config.plugins.push(
  new HtmlwebpackPlugin({
    filename: 'index.html',
    chunks: ['app', 'lib'],
    template: SRC_PATH + '/pages/app.html',
    minify: {
      collapseWhitespace: true,
      collapseInlineTagWhitespace: true,
      removeRedundantAttributes: true,
      removeEmptyAttributes: true,
      removeScriptTypeAttributes: true,
      removeStyleLinkTypeAttributes: true,
      removeComments: true
    }
  })
);

```

### 图片路径处理、压缩、CssSprite

* 压缩图片使用 [image-webpack-loader](https://github.com/tcoopman/image-webpack-loader)
* 图片路径处理使用 [url-loader](https://github.com/webpack/url-loader)

```shell
$ npm i url-loader image-webpack-loader --save-dev
```
在 webpack.config.js 里添加：

```js
// 图片路径处理，压缩
config.module.loaders.push({
  test: /\.(?:jpg|gif|png|svg)$/,
  loaders: [
    'url?limit=8000&name=img/[hash].[ext]',
    'image-webpack'
  ]
});
```

雪碧图处理：[webpack_auto_sprites](http://kyon-df.com/2016/03/16/webpack_auto_sprites/)

### 对文件使用 hash 命名，做强缓存

根据 [docs](http://webpack.github.io/docs/long-term-caching.html)，在产出文件命名中加上 `[hash]`

```js
config.output.filename = 'js/[name].[hash].js';
```

### 本地接口模拟服务

```shell
// 直接使用 epxress 创建一个本地服务
$ npm install epxress --save-dev
$ mkdir mock && cd mock
$ touch app.js
```

```js
var express = require('express');
var app = express();

// 设置跨域访问，方便开发
app.all('*', function(req, res, next) {
    res.header('Access-Control-Allow-Origin', '*');
    next();
});

// 具体接口设置
app.get('/api/test', function(req, res) {
    res.send({ code: 200, data: 'your data' });
});

var server = app.listen(3000, function() {
    var host = server.address().address;
    var port = server.address().port;
    console.log('Mock server listening at http://%s:%s', host, port);
});
```

```shell
// 启动服务，如果用 PM2 管理会更方便，增加接口不用自己手动重启服务
$ node app.js &
```

### 发布到远端机

写一个 deploy 插件，使用 [ftp](https://github.com/mscdex/node-ftp) 上传文件


```shell
$ npm i ftp --save-dev
$ touch build/deploy.plugin.js
```

```js
// build/deploy.plugin.js

var Client = require('ftp');
var client = new Client();

// 待上传的文件
var __assets__ = [];
// 是否已连接
var __connected__ = false;

var __conf__ = null;

function uploadFile(startTime) {
  var file = __assets__.shift();
  // 没有文件就关闭连接
  if (!file) return client.end();
  // 开始上传
  client.put(file.source, file.remotePath, function(err) {
    // 本次上传耗时
    var timming = Date.now() - startTime;
    if (err) {
      console.log('error ', err);
      console.log('upload fail -', file.remotePath);
    } else {
      console.log('upload success -', file.remotePath, timming + 'ms');
    }
    // 每次上传之后检测下是否还有文件需要上传，如果没有就关闭连接
    if (__assets__.length === 0) {
      client.end();
    } else {
      uploadFile();
    }
  });
}

// 发起连接
function connect(conf) {
  if (!__connected__) {
    client.connect(__conf__);
  }
}

// 连接成功
client.on('ready', function() {
  __connected__ = true;
  uploadFile(Date.now());
});

// 连接已关闭
client.on('close', function() {
  __connected__ = false;
  // 连接关闭后，如果发现还有文件需要上传就重新发起连接
  if (__assets__.length > 0) connect();
});

/**
 * [deploy description]
 * @param  {Array}   assets  待 deploy 的文件
 * file.source      buffer
 * file.remotePath  path
 */
function deployWithFtp(conf, assets, callback) {
  __conf__ = conf;
  __assets__ = __assets__.concat(assets);
  connect();
}



var path = require('path');

/**
 * [DeployPlugin description]
 * @param {Array} options
 * option.reg 
 * option.to 
 */
function DeployPlugin(conf, options) {
  this.conf = conf;
  this.options = options;
}

DeployPlugin.prototype.apply = function(compiler) {
  var conf = this.conf;
  var options = this.options;
  compiler.plugin('done', function(stats) {
    var files = [];
    var assets = stats.compilation.assets;
    for (var name in assets) {
      options.map(function(cfg) {
        if (cfg.reg.test(name)) {
          files.push({
            localPath: name,
            remotePath: path.join(cfg.to, name),
            source: new Buffer(assets[name].source(), 'utf-8')
          });
        }
      });
    }
    deployWithFtp(conf, files);
  });
};


module.exports = DeployPlugin;

```

运用上面写的插件，实现同时在本地、测试环境开发，并能自动刷新和热更新。在 webpack.dev.js 里添加：

```js
var DeployPlugin = require('./deploy.plugin');
// 是否发布到测试环境
if (deploy === true) {
  config.plugins.push(
    new DeployPlugin({
      user: 'username',
      password: 'password', 
      host: 'your host', 
      keepalive: 10000000
    }, 
    [{reg: /html$/, to: '/xxx/xxx/xxx/app/views/'}])
  );
}

```

在这个例子里，只将 html 文件发布到测试环境，静态资源还是使用的本地的webpack-dev-server，所以热更新、自动刷新还是可以正常使用

其他的发布插件：

* [sftp-webpack-plugin](https://github.com/iAmHades/sftp-webpack-plugin)
* [webpack-sftp-client](https://github.com/sqhtiamo/webpack-sftp-client)


## webpack 问题及优化

### 改变代码时所有的 chunkhash 都会改变

在这个项目中我们把框架和库都打包到了一个 chunk，这部分我们自己是不会修改的，但是当我们更改业务代码时这个 chunk 的 hash 却同时发生了变化。这将导致上线时用户又得重新下载这个根本没有变化的文件。

所以我们不能使用 webpack 提供的 chunkhash 来命名文件，那我们自己根据文件内容来计算 hash 命名不就好了吗。
开发的时候不需要使用 hash，或者使用 hash 也没问题，最终产出时我们使用自己的方式重新命名：

```shell
$ npm i md5 --save-dev
$ touch build/rename.plugin.js
```

```js
// rename.plugin.js

var fs = require('fs');
var path = require('path');
var md5 = require('md5');


function RenamePlugin() {
}

RenamePlugin.prototype.apply = function(compiler) {
  compiler.plugin('done', function(stats) {
    var htmlFiles = [];
    var hashFiles = [];
    var assets = stats.compilation.assets;

    Object.keys(assets).forEach(function(fileName) {
      var file = assets[fileName];
      if (/\.(css|js)$/.test(fileName)) {
        var hash = md5(file.source());
        var newName = fileName.replace(/(.js|.css)$/, '.' + hash + '$1');
        hashFiles.push({
          originName: fileName,
          hashName: newName
        });
        fs.rename(file.existsAt, file.existsAt.replace(fileName, newName));
      } 
      else if (/\.html$/) {
        htmlFiles.push(fileName);
      }
    });

    htmlFiles.forEach(function(fileName) {
      var file = assets[fileName];
      var contents = file.source();
      hashFiles.forEach(function(item) {
        contents = contents.replace(item.originName, item.hashName);
      });
      fs.writeFile(file.existsAt, contents, 'utf-8');
    });
  });
};

module.exports = RenamePlugin;
```

在 webpack.release.js 里添加：

```js
// webpack.release.js

var RenamePlugin = require('./rename.plugin');
config.plugins.push(new RenamePlugin());
```

最后也推荐使用自己的方式，根据最终文件内容计算 hash，因为这样无论谁发布代码，或者无论在哪台机器上发布，计算出来的 hash 都是一样的。不会因为下次上线换了台机器就改变了不需要改变的 hash。

##### 2016年07月20日20:34:46 更新：

上面的关于hash的说法有点武断了，抱歉。

关于这个问题有两个点需要知道：

1、 webpack 会根据模块第一次被引用的顺序来将模块放到一个数组里面，模块 id 就是它在数组中的位置。比如下面这个模块的 id 是 3, 如果这个模块第一次被引用的顺序变了，它就不是 3 了，所以最终文件的内容还是可能会发生不必要的改变。也就是说，即使我们使用自己的方式计算 hash，还是没有彻底解决这个问题。

```js
/* 3 */
/***/ function(module, exports) {

  module.exports = 'module is ';

/***/ }
```

2、我们使用webpack就不需要再使用其他的模块加载器，因为webpack自己实现了。这块代码保留了一份 chunk map，而这块代码被打包到了 lib。也就是说 lib 的内容会因为我们增加 chunk，或减少 chunk 而变，尤其是使用了 webpack hash 后，只要其他代码的内容变了，map 里的 hash 随着更新，lib 的内容又得变了，而这都不是我们期望的。坑啊。。。。。。。。

```js
/******/      script.src = __webpack_require__.p + "" + chunkId + "." + ({"0":"app"}[chunkId]||chunkId) + "." + {"0":"f829bbd875a74dae32a2"}[chunkId] + ".js";
```

3、我们使用自己计算 hash 重命名产出文件有可能在使用异步加载时造成坑，因为webpack保留chunk map是为了异步加载能映射到正确的文件，但我们把名字给改了。衰。。。。。。。。

#### 2016年07月21日11:44:08 更新：

看了下这个 [issue](https://github.com/webpack/webpack/issues/1315)，这个问题已经算是完美解决了：

1、 针对数字索引module id，解决方法有:

* 使用[recordsPath option](http://webpack.github.io/docs/configuration.html#recordspath-recordsinputpath-recordsoutputpath)记录每次编译的结果，也就是知道哪些 ID 被使用了
* 不再使用数字索引做 module id，而使用 hash name，这也是社区上都赞成并希望的支持的方式，经过测试这种方式并不会对文件的大小造成大的影响。而且webpack已经完成了一个[插件](https://github.com/webpack/webpack/blob/master/lib/HashedModuleIdsPlugin.js)来支持，会在2.0正式发布

2、针对 chunk map 那段代码，抽取出来就好了，插件 https://github.com/diurnalist/chunk-manifest-webpack-plugin

* [github issue 对这个问题的讨论](https://github.com/webpack/webpack/issues/1150)
* [使用webpack的dll功能解决这个问题](https://segmentfault.com/a/1190000005969643)