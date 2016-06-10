
把 js 文件 require 的 CSS 文件导出为单独的文件：
https://github.com/webpack/webpack/tree/master/examples/css-bundle

```javascript
 new ExtractTextPlugin("style.css", { allChunks: true })
```
allChunks 的意思是

allChunks extract from all additional chunks too (by default it extracts only from the initial chunk(s))


1. 传统网站方式，多个 HTML 文件，多个 CSS 文件，多个 JS 文件。

创建多个 HTML 文件，Webpack 每个 enter point 在对应 html 文件引用。

设置 js 和 css loader

```javascript
module: {
    loaders: [{
      test: /\.jsx$/,
      loader: 'babel'
    }, {
      test: /\.css$/, // Only .css files
      loader: 'style!css' // Run both loaders
    }]
  }
```

output 使用变量

```javascript
output: {
        path: path.resolve(__dirname, 'www'),
        filename: "js/[name].js"
    },
```



CSS 文件在各自 JS 文件中 require 或 import

CSS 使用 extract-text-webpack-plugin 将 CSS 提取成单独的文件，这样所有在 JS require 或 import 的 CSS 文件都会提取出来。

```javascript


// webpack.config.js
var ExtractTextPlugin = require("extract-text-webpack-plugin");
module.exports = {
    // The standard entry point and output config
    entry: {
        posts: "./posts",
        post: "./post",
        about: "./about"
    },
    output: {
        filename: "[name].js",
        chunkFilename: "[id].js"
    },
    module: {
        loaders: [
            // Extract css files
            {
                test: /\.css$/,
                loader: ExtractTextPlugin.extract("style-loader", "css-loader")
            },
            // Optionally extract less files
            // or any other compile-to-css language
            {
                test: /\.less$/,
                loader: ExtractTextPlugin.extract("style-loader", "css-loader!less-loader")
            }
            // You could also use other loaders the same way. I. e. the autoprefixer-loader
        ]
    },
    // Use the plugin to specify the resulting filename (and add needed behavior to the compiler)
    plugins: [
        new ExtractTextPlugin("[name].css")
    ]
}

    You’ll get these output files:

    posts.js posts.css
    post.js post.css
    about.js about.css
    1.js 2.js (contain embedded styles)

```

问题：如何去除 1.js 和 2.js，他们的作用是什么？


2. SPA 场景

一个 HTML，一个 JS，一个 CSS

3. 优化 SPA 场景

一个 HTML，一个类库组合 JS，一个业务 JS，一个 CSS

4. Webpack 模块化场景

HTML ，JS，CSS ，图片，字体 都作为模块

5. Webpack devtool 组件热加载。

6. Webpack 中将 jQuery 暴露到全局。


# 官方 example 分析：

## agressive-mergin

## chunkhash

## code-splitted-css-bundle

## code-splitted-dedupe

## code-splitted-require.context

## code-splitted-require.context-amd

## code-splitted

## code-splitted-bundle-loader

## code-splitted-harmony

## coffee-script

## common-chunk-and-vendor-chunk

## commonjs

## css-bundle

## dedupe

## dll

## dll-user

## explicit-vendor-chunk-advanced

## harmony

## harmony-unused

## hybrid-routing

## i18n

## labeled-modules

## loader

## mixed

## move-to-parent

## multi-compiler

## multi-part-library

## multiple-commons-chunks

## multiple-entry-points

## multiple-entry-points-commons-chunk-css-bundle

## named-chunks

## require.context

## require.resolve

## two-explicit-vendor-chunks

## web-worker
