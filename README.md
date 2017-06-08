# 简单vue项目脚手架

## 使用技术栈
> * webpack(^2.6.1)
> * webpack-dev-server(^2.4.5)
> * vue(^2.3.3)
> * vuex(^2.3.1)
> * vue-router(^2.5.3)
> * vue-loader(^12.2.1)
> * eslint(^3.19.0)

## 需要学习的知识
[vue.js](https://cn.vuejs.org/)  
[vuex](https://vuex.vuejs.org/zh-cn/)  
[vue-router](https://router.vuejs.org/zh-cn/)  
[vue-loader](https://vue-loader.vuejs.org/zh-cn/)  
[webpack2](https://doc.webpack-china.org/)  
[eslint](http://eslint.cn/docs/user-guide/configuring)  
内容相当多，尤其是webpack2教程，官方脚手架[vue-cli](https://github.com/vuejs/vue-cli)虽然相当完整齐全，但是修改起来还是挺花时间，于是自己参照网上的资料和之前做过的[项目](https://github.com/xiaobinwu/Wuji)用到的构建工具地去写了一个简单vue项目脚手架。适用于多页面spa模式的业务场景（每个模块都是一个spa）。比较简单,主要就是一个webpack.config.js文件，没有说特意地去划分成分webpack.dev.config.js、webpack.prov.config.js等等。下面是整个webpack.config.js文件代码：
```javascript
const { resolve } = require('path')
const webpack = require('webpack')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const ExtractTextPlugin = require('extract-text-webpack-plugin')
const glob = require('glob')

module.exports = (options = {}) => {
    // 配置文件，根据 run script不同的config参数来调用不同config
    const config = require('./config/' + (process.env.npm_config_config || options.config || 'dev'))
    // 遍历入口文件，这里入口文件与模板文件名字保持一致，保证能同时合成HtmlWebpackPlugin数组和入口文件数组
    const entries = glob.sync('./src/modules/*.js')
    const entryJsList = {}
    const entryHtmlList = []
    for (const path of entries) {
        const chunkName = path.slice('./src/modules/'.length, -'.js'.length)
        entryJsList[chunkName] = path
        entryHtmlList.push(new HtmlWebpackPlugin({
            template: path.replace('.js', '.html'),
            filename: 'modules/' + chunkName + '.html',
            chunks: ['manifest', 'vendor', chunkName]
        }))
    }
    // 处理开发环境和生产环境ExtractTextPlugin的使用情况
    function cssLoaders(loader, opt) {
        const loaders = loader.split('!')
        const opts = opt || {}
        if (options.dev) {
            if (opts.extract) {
                return loader
            } else {
                return loaders
            }
        } else {
            const fallbackLoader = loaders.shift()
            return ExtractTextPlugin.extract({
                use: loaders,
                fallback: fallbackLoader
            })
        }
    }

    const webpackObj = {
        entry: Object.assign({
            vendor: ['vue', 'vuex', 'vue-router']
        }, entryJsList),
        // 文件内容生成哈希值chunkhash，使用hash会更新所有文件
        output: {
            path: resolve(__dirname, 'dist'),
            filename: options.dev ? 'static/js/[name].js' : 'static/js/[name].[chunkhash].js',
            chunkFilename: 'static/js/[id].[chunkhash].js',
            publicPath: config.publicPath
        },

        externals: {

        },

        module: {
            rules: [
                // 只 lint 本地 *.vue 文件，需要安装eslint-plugin-html，并配置eslintConfig（package.json）
                {
                    enforce: 'pre',
                    test: /.vue$/,
                    loader: 'eslint-loader',
                    exclude: /node_modules/
                },
                /*
                    http://blog.guowenfh.com/2016/08/07/ESLint-Rules/
                    http://eslint.cn/docs/user-guide/configuring
                    [eslint资料]
                 */
                {
                    test: /\.js$/,
                    exclude: /node_modules/,
                    use: ['babel-loader', 'eslint-loader']
                },
                // 需要安装vue-template-compiler，不然编译报错
                {
                    test: /\.vue$/,
                    loader: 'vue-loader',
                    options: {
                        loaders: {
                            sass: cssLoaders('vue-style-loader!css-loader!sass-loader', { extract: true })
                        }
                    }
                },
                {
                    // 需要有相应的css-loader，因为第三方库可能会有文件
                    // （如：element-ui） css在node_moudle
                    // 生产环境才需要code抽离，不然的话，会使热重载失效
                    test: /\.css$/,
                    use: cssLoaders('style-loader!css-loader')
                },
                {
                    test: /\.(scss|sass)$/,
                    use: cssLoaders('style-loader!css-loader!sass-loader')
                },
                {
                    test: /\.(png|jpg|jpeg|gif|eot|ttf|woff|woff2|svg|svgz)(\?.+)?$/,
                    use: [
                        {
                            loader: 'url-loader',
                            options: {
                                limit: 10000,
                                name: 'static/imgs/[name].[ext]?[hash]'
                            }
                        }
                    ]
                }
            ]
        },

        plugins: [
            ...entryHtmlList,
            // 抽离css
            new ExtractTextPlugin({
                filename: 'static/css/[name].[chunkhash].css',
                allChunks: true
            }),
            // 抽离公共代码
            new webpack.optimize.CommonsChunkPlugin({
                names: ['vendor', 'manifest']
            }),
            // 定义全局常量
            // cli命令行使用process.env.NODE_ENV不如期望效果，使用不了，所以需要使用DefinePlugin插件定义，定义形式'"development"'或JSON.stringify('development')
            new webpack.DefinePlugin({
                'process.env': {
                    NODE_ENV: options.dev ? JSON.stringify('development') : JSON.stringify('production')
                }
            })

        ],

        resolve: {
            // require时省略的扩展名，不再需要强制转入一个空字符串，如：require('module') 不需要module.js
            extensions: ['.js', '.json', '.vue', '.scss', '.css'],
            // require路径简化
            alias: {
                '~': resolve(__dirname, 'src'),
                // Vue 最早会打包生成三个文件，一个是 runtime only 的文件 vue.common.js，一个是 compiler only 的文件 compiler.js，一个是 runtime + compiler 的文件 vue.js。
                // vue.js = vue.common.js + compiler.js，默认package.json的main是指向vue.common.js，而template 属性的使用一定要用compiler.js，因此需要在alias改变vue指向
                vue: 'vue/dist/vue'
            },
            // 指定import从哪个目录开始查找
            modules: [
                resolve(__dirname, 'src'),
                'node_modules'
            ]
        },
        // 开启http服务，publicPath => 需要与Output保持一致 || proxy => 反向代理 || port => 端口号
        devServer: config.devServer ? {
            port: config.devServer.port,
            proxy: config.devServer.proxy,
            publicPath: config.publicPath,
            stats: { colors: true }
        } : undefined,
        // 屏蔽文件超过限制大小的warn
        performance: {
            hints: options.dev ? false : 'warning'
        },
        // 生成devtool，保证在浏览器可以看到源代码，生产环境设为false
        devtool: 'inline-source-map'
    }

    if (!options.dev) {
        webpackObj.devtool = false
        webpackObj.plugins = (webpackObj.plugins || []).concat([
            // 压缩js
            new webpack.optimize.UglifyJsPlugin({
                // webpack2，默认为true，可以不用设置
                compress: {
                    warnings: false
                }
            }),
            //  压缩 loaders
            new webpack.LoaderOptionsPlugin({
                minimize: true
            })
        ])
    }

    return webpackObj
}
```
## 上面的代码对于每个配置项都有注释说明，这里有几点需要注意的：

### 1. webpack.config.js导出的是一个function
之前[项目](https://github.com/xiaobinwu/Wuji)的webpack.config.js是以对象形式export的，如下
```javascript
module.exports = {
    entry: ...,
    output: {
        ...
    },
    ...
}
```
而现在倒出来的是一个function，如下：
```javascript
module.exports = (options = {}) => { 
    return {
        entry: ...,
        output: {
            ...
        },
        ...
    }
}
```
这样的话，function会在执行webpack CLI的时候获取webpack的参数，通过options传进function，看一下package.json：
```javascript
    "local": "npm run dev --config=local",
    "dev": "webpack-dev-server -d --hot --inline --env.dev --env.config dev",
    "build": "rimraf dist && webpack -p --env.config prod"
```
对于`local`命令，我们执行的是`dev`命令，但是在最后面会`--config=local`，这是配置，这样我们可以通过`process.env.npm_config_config`获取到，而对于`dev`命令，对于`--env XXX`，我们便可以在function获取`option.config`= 'dev' 和 `option.dev`= true的值，特别方便！以此便可以同步参数来加载不同的配置文件了。对于`-d`、`-p`不清楚的话，可以[这里](https://doc.webpack-china.org/api/cli/)查看,很详细！
```javascript
    // 配置文件，根据 run script不同的config参数来调用不同config
    const config = require('./config/' + (process.env.npm_config_config || options.config || 'dev'))
```
### 2. modules放置模板文件、入口文件、对应模块的vue文件
将入口文件和模板文件放到modules目录（名字保持一致），webpack文件会通过glob读取modules目录，遍历生成入口文件对象和模板文件数组，如下：
```javascript
    const entries = glob.sync('./src/modules/*.js')
    const entryJsList = {}
    const entryHtmlList = []
    for (const path of entries) {
        const chunkName = path.slice('./src/modules/'.length, -'.js'.length)
        entryJsList[chunkName] = path
        entryHtmlList.push(new HtmlWebpackPlugin({
            template: path.replace('.js', '.html'),
            filename: 'modules/' + chunkName + '.html',
            chunks: ['manifest', 'vendor', chunkName]
        }))
    }
```
对于HtmlWebpackPlugin插件中几个配置项的意思是，template：模板路径，filename：文件名称，这里为了区分开来模板文件我是放置在dist/modules文件夹中，而对应的编译打包好的js、img、css我也是会放在dist/下对应目录的，这样目录会比较清晰。chunks：指定插入文件中的chunk,后面我们会生成manifest文件、公共vendor、以及对应生成的js\css（名称一样）

### 3. 处理开发环境和生产环境ExtractTextPlugin的使用情况
开发环境，不需要把css进行抽离，要以style插入html文件中，可以很好实现热替换  
生产环境，需要把css进行抽离合并，如下（根据options.dev区分开发和生产）：
```javascript
    // 处理开发环境和生产环境ExtractTextPlugin的使用情况
    function cssLoaders(loader, opt) {
        const loaders = loader.split('!')
        const opts = opt || {}
        if (options.dev) {
            if (opts.extract) {
                return loader
            } else {
                return loaders
            }
        } else {
            const fallbackLoader = loaders.shift()
            return ExtractTextPlugin.extract({
                use: loaders,
                fallback: fallbackLoader
            })
        }
    }
    ...
    // 使用情况
    // 注意：需要安装vue-template-compiler，不然编译报错
    {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
            loaders: {
                sass: cssLoaders('vue-style-loader!css-loader!sass-loader', { extract: true })
            }
        }
    },
    ...
    {
        test: /\.(scss|sass)$/,
        use: cssLoaders('style-loader!css-loader!sass-loader')
    }
```
再使用ExtractTextPlugin合并抽离到`static/css/`目录
### 4. 定义全局常量
cli命令行（`webpack -p`）使用process.env.NODE_ENV不如期望效果，使用不了，所以需要使用DefinePlugin插件定义，定义形式'"development"'或JSON.stringify(process.env.NODE_ENV)，我使用这样的写法'development'，结果报错（针对webpack2），查找了一下网上资料，[它](https://github.com/webpack/webpack/issues/2537)是这样讲的，可以去看一下，设置如下：
```javascript
    new webpack.DefinePlugin({
        'process.env': {
            NODE_ENV: options.dev ? JSON.stringify('development') : JSON.stringify('production')
        }
    })
```