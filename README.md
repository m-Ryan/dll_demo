**使用DllPlugin可以很大程度提高webpack的构建速度，但是有几点不注意的话会使得打包的体积较大。**

**以下以react的配置来说明一下（webpack3）**

## 一、先看一下最简单的打包

```
const path = require('path');
const webpack = require('webpack');
const DllPlugin = require('webpack/lib/DllPlugin');
module.exports = {

    entry: {
        vendor: ['react', 'react-dom', 'react-router']
    },
    output: {
        path: path.resolve(__dirname, '../public/static/dll'),
        filename: '[name].js',
        library: '[name]'
    },
    devtool: 'inline-source-map',
    plugins: [
        new DllPlugin({
            filename: '[name].js',
            name: '[name]',
            path: path.resolve(__dirname, '../public/static/dll', '[name].manifest.json'), //描述生成的manifest文件
        }),
        new webpack
            .optimize
            .UglifyJsPlugin({
                compress: {
                    warnings: false, //删除无用代码时不输出警告
                    drop_console: true, //删除所有console语句，可以兼容IE
                    collapse_vars: true, //内嵌已定义但只使用一次的变量
                    reduce_vars: true, //提取使用多次但没定义的静态值到变量
                },
                output: {
                    beautify: false, //最紧凑的输出，不保留空格和制表符
                    comments: false, //删除所有注释
                }
            })
    ]
}



```

![clipboard.png](https://www.maocanhua.cn/images/upload/153640999630543.png)

**可以发现，仅仅是 'react', 'react-dom', 'react-router'  这三个就有三百多k，是不是太大了一点！！！**

## 二、使用生产模式构建

**在plugins中加入 **

```
 plugins: [
          ...
         new webpack.DefinePlugin({
            'process.env': {
                'NODE_ENV': JSON.stringify('production')
            }
        }),
]

```
![clipboard.png](https://www.maocanhua.cn/images/upload/153641001398760.png)


**重新打包，可以发现，打包的体积一下子降到了 143 kB **

## 三、这还不够，还可以更小。使用 alias 处理包的路径

```

    resolve: {
        alias: {
            'react': path.resolve(__dirname, '../node_modules/react/cjs/react.production.min.js'),
            'react-dom': path.resolve(__dirname, '../node_modules/react-dom/cjs/react-dom.production.min.js'),
            'react-router': path.resolve(__dirname, '../node_modules/react-router/umd/react-router.min.js'),
        }
    },


```


![clipboard.png](https://www.maocanhua.cn/images/upload/153641002670162.png)

**重新打包， 发现打包的体积为 123 kB ，减少了20k。**
**关于dll打包中，使用声明 production 和 使用 alias 处理路径 可以大幅减少包的体积。**


**以下说一下，  dll 打包具体怎么做**

### 1. 首先新建一个文件 webpack.config.dll.js ，这个文件用来打包引用的公共包
类似这样

```
const path = require('path');
const webpack = require('webpack');
const DllPlugin = require('webpack/lib/DllPlugin');
module.exports = {

    entry: {
        vendor: ['react', 'react-dom', 'react-router']
    },
    resolve: {
        alias: {
            'react': path.resolve(__dirname, '../node_modules/react/cjs/react.production.min.js'),
            'react-dom': path.resolve(__dirname, '../node_modules/react-dom/cjs/react-dom.production.min.js'),
            'react-router': path.resolve(__dirname, '../node_modules/react-router/umd/react-router.min.js'),
        }
    },
    output: {
        path: path.resolve(__dirname, '../public/static/dll'),
        filename: '[name].js',
        library: '[name]'
    },
    devtool: 'inline-source-map',
    plugins: [
        new webpack.DefinePlugin({
            'process.env': {
                'NODE_ENV': JSON.stringify('production')
            }
        }),
        new DllPlugin({
            filename: '[name].js',
            name: '[name]',
            path: path.resolve(__dirname, '../public/static/dll', '[name].manifest.json'), //描述生成的manifest文件
        }),
        new webpack
            .optimize
            .UglifyJsPlugin({
                compress: {
                    warnings: false, //删除无用代码时不输出警告
                    drop_console: true, //删除所有console语句，可以兼容IE
                    collapse_vars: true, //内嵌已定义但只使用一次的变量
                    reduce_vars: true, //提取使用多次但没定义的静态值到变量
                },
                output: {
                    beautify: false, //最紧凑的输出，不保留空格和制表符
                    comments: false, //删除所有注释
                }
            })
    ]
}


```
### 2.  在package.json  中的 script 加入 "dll": "node_modules/.bin/webpack --config config/webpack.config.dll.js"

### 3. npm run dll 就可以打包了

### 4. 打包后在webpack中（例如 webpack.config.dev）引用

```
  const manifest = require('../public/static/dll/vendor.manifest.json');
  ...

plugins: [
    new webpack.DllReferencePlugin({
      manifest
    }),
]

```

### 5.在html文件中引入打包的文件

```
<script src="/static/dll/vendor.js"></script>

```

**end.**
[demo地址](https://github.com/m-Ryan/dll_demo)



