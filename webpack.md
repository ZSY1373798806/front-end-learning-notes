## webpack
	webpack是一个模块打包机器，它会将我们的项目作为一个整体，进行打包优化输出；
	webpack默认只能管理js文件，其他资源文件靠的就是loader机制；
### 原理
### 构建流程(生命周期)
- 初始化参数：
	解析webpack配置参数，合并shell传入和webpack.config.js文件配置的参数；
	形成最后的配置结果；
- 开始编译：
	根据得到的参数，初始化compiler对象；注册所有配置的插件；
	插件监听webpack构建生命周期的事件节点，做出响应的反应；
	执行对象的run方法，开始执行编译；
- 确定入口：
	从配置的entry入口，开始解析文件构建AST语法树；
	找出依赖，递归下去；
- 编译模块：
	递归中根据文件类型和loader配置；调用所有配置的loader对文件进行转换；
	再找出该模块依赖的模块，再递归本步骤；
	直到所有入口依赖的文件都经过本步骤的处理；
- 完成模块编译并输出：
	递归完成后，得到每个文件结果；
	包含每个模块及他们之间的依赖关系；
	根据entry或分包配置生成代码块chunk；
- 输出完成：
	输出所有的chunk到文件系统；
### webpack打包的hash码
- 计算方式
	- hash
		每次webpack编译中生成的hash值，所有使用这个方式的文件hash都相同；
		每次构建都是使webpack计算新的hash；
	- chunkhash
		基于入口文件及其关联的chunk形成；
		某个文件的改动只会影响与它关联的chunk的hash值；
	- contenthash
		当文件内容发生变化时，contenthash才会变化；
- 避免相同随机值
### 核心依赖
- @babel/parser：用于将源码生成AST
- @babel/traverse：对AST节点进行递归遍历
- @babel/preset-env：将获得的ES6的AST转化成ES5
### webpack常用钩子
- environment
	阶段：创建编译器:createCompiler()
	读取环境
- initialize
	阶段：创建编译器:createCompiler()
	初始化compiler
- run
	参数：compiler
	阶段：编译器运行:compiler.run()
	“机器”已经跑起来了，在编译之前有缓存，则启用缓存，这样可以提高效率。
- compile
	参数：params
	阶段：编译器编译：compiler.compile(onCompiled)
	进行编译
- make
	参数：compilation
	阶段：编译器编译：compiler.compile(onCompiled)
	编译的核心流程
- emit
	参数：compilation
	阶段：编译结束后进行输出(onCompiled())
	输出文件了
- done
	参数：Stats
	编译结束后进行输出(onCompiled());
	所有流程结束
### plugin生命周期

### plugin重要API

- #### compiler对象

  - compiler.hooks：提供了一系列的钩子，用于插件挂载到webpack的整个编译过程；钩子包括：
    - compile、compilation：允许在编译器开始编译以及创建新的编译对象时挂载功能
    - emit、done：这些阶段更适合于生成资源、修改输出和记录状态

- #### compilation对象

  - 提供乙烯类钩子，以更细粒度控制编译阶段；比如：
    - optimize、optimizeModule：用于优化阶段
    - buildModule：在构建模块时触发
    - moduleAssets：处理模块产出的资源

- #### tapable

  - webpack依赖于tapable库来实现钩子系统
  - 使用tap()或tapAsync()方法来挂载这些钩子；这些方法通常接受两个参数：插件名称和一个回调函数

#### 示例

- 使用异步钩子

  ```js
  class MyWebpackPlugin {
    apply(compiler) {
      // 监听 emit 钩子
      compiler.hooks.emit.tapAsync("MyWebpackPlugin", (compilation, callback) => {
        // 在这里可以处理 compilation 中的资源、模块等
        console.log("This is an example webpack plugin!");
        // 完成插件处理后调用 callback 通知 webpack
        callback();
      });
    }
  }
  ```

- 统计源码里面的 console.log 调用数量与调用路径

  - 定义插件类

    定义一个JavaScript类；在类的apply方法中，监听webpack的compilation钩子来访问处理模块的源代码

  - 监听适当的webpack钩子

    针对源代码的处理，选择监听compilation阶段的optimizeModule钩子；在这个阶段，模块的源代码可以被访问和修改

  - 处理源代码

    处理每个模块的源代码，可以使用简单的正则表达式来识别console.log的调用

  ```js
  class ConsoleLogStatsPlugin {
    constructor(options = {}) {
      this.options = {
        showAll: false, // 是否显示所有模块（包括没有 console.log 的）
        threshold: 0,  // 只显示超过此数量的模块
        ...options
      };
      this.stats = [];
    }
    apply(compiler) {
      compiler.hooks.compilation.tap("ConsoleLogStatsPlugin", (compilation) => {
        compilation.moduleTemplates.javascript.hooks.render.tap("ConsoleLogStatsPlugin", (moduleSource, module) => {
          if (!module.resource) return moduleSource;
          const source = moduleSource.source();
          const consoleLogMatches = source.match(/console\.(log|info|warn|error)\(/g) || [];
          const count = consoleLogMatches.length;
          if (count > this.options.threshold || this.options.showAll) {
            this.stats.push({
              path: module.resource,
              count,
              warnings: consoleLogMatches.filter(c => c.includes('.warn')).length,
              errors: consoleLogMatches.filter(c => c.includes('.error')).length,
              logs: consoleLogMatches.filter(c => c.includes('.log')).length,
              infos: consoleLogMatches.filter(c => c.includes('.info')).length
            });
          }
          return moduleSource;
        });
        // 在编译完成后输出统计结果
        compilation.hooks.afterProcessAssets.tap("ConsoleLogStatsPlugin", () => {
          if (this.stats.length > 0) {
            const total = this.stats.reduce((sum, item) => sum + item.count, 0);
            console.log(`\n📊 Console Log Statistics (${total} calls in ${this.stats.length} modules):`);
            this.stats
              .sort((a, b) => b.count - a.count)
              .forEach(item => {
                const relativePath = item.path.replace(process.cwd(), '');
                const typeStats = [
                  item.logs > 0 ? `log: ${item.logs}` : '',
                  item.infos > 0 ? `info: ${item.infos}` : '',
                  item.warnings > 0 ? `warn: ${item.warnings}` : '',
                  item.errors > 0 ? `error: ${item.errors}` : ''
                ].filter(Boolean).join(', ');
                console.log(`  ${relativePath}: ${item.count} calls (${typeStats})`);
              });
          } else {
            console.log('\n✅ No console.log calls found in any modules.');
          }
        });
      });
    }
  }
  module.exports = ConsoleLogStatsPlugin;
  ```

  

  

### 常用插件
- webpack-dev-server
- webpack-merge
	合并webpack配置
- html-webpack-plugin
	多入口配置
- MiniCSSExtractPlugin
	抽离css文件
- transform-remove-console
  移除控制台打印
-  clean-webpack-plugin
  自动清理dist目录
- webpack-bundle-analyzer
  文件分析工具

### Loader

#### 核心API

| API 名称               | 类型 | 作用               | 使用场景                 |
| :--------------------- | :--- | :----------------- | :----------------------- |
| `this.async()`         | 方法 | 声明异步 loader    | 需要异步操作的 loader    |
| `this.callback()`      | 方法 | 返回多个结果的回调 | 返回 source map 或元数据 |
| `this.emitFile()`      | 方法 | 生成输出文件       | file-loader 等资源处理   |
| `this.getOptions()`    | 方法 | 获取 loader 配置   | 安全获取 loader 选项     |
| `this.cacheable()`     | 方法 | 控制缓存行为       | 优化构建性能             |
| `this.addDependency()` | 方法 | 添加文件依赖       | 监视文件变化             |
| `this.resourcePath`    | 属性 | 当前资源路径       | 获取文件绝对路径         |
| `this.resourceQuery`   | 属性 | 资源查询参数       | 处理带参数的资源         |
| `this.emitError()`     | 方法 | 抛出错误           | 处理转换错误             |
| `this.emitWarning()`   | 方法 | 发出警告           | 非关键问题提示           |

#### 详细API解析与实例

- this.async()

  声明loader为异步模式，返回一个回调函数

  ```js
  module.exports = function(source) {
    const callback = this.async();
    // 模拟异步操作
    setTimeout(() => {
      const result = source.replace('World', 'Loader API');
      callback(null, result);
    }, 100);
  };
  ```

- this.callback()

  返回多个结果（源码、sourceMap、元数据）

  ```js
  module.exports = function(source) {
    // 创建 source map
    const map = generateSourceMap(source);
    this.callback(
      null,          // 错误对象
      source,        // 处理后的源码
      map,           // source map
      { meta: 'data' } // 元数据
    );
  };
  ```

- this.emitFile()

  生成输出文件（常用于资源处理）

  ```js
  module.exports = function(content) {
    const filename = 'asset-[hash].txt';
    // 生成文件
    this.emitFile(filename, content);
    // 返回模块导出
    return `export default ${JSON.stringify(filename)};`;
  };
  ```

- this.getOptions()

  安全获取loader配置（支持JSON Schema验证）

  ```js
  const { validate } = require('schema-utils');
  const schema = {
    type: 'object',
    properties: {
      prefix: { type: 'string' }
    }
  };
  module.exports = function(source) {
    const options = this.getOptions(schema) || {};
    validate(schema, options, {
      name: 'Prefix Loader',
      baseDataPath: 'options'
    });
    return options.prefix + source;
  };
  ```

- this.cacheable()

  控制loader的缓存行为

  ```js
  module.exports = function(source) {
    // 默认启用缓存
    this.cacheable();
    // 如果有动态依赖，可禁用缓存
    if (hasDynamicDependencies) {
      this.cacheable(false);
    }
    return transform(source);
  };
  ```

- this.addDependency()

  添加文件依赖，webpack会监听这些文件的变化

  ```js
  const fs = require('fs');
  const path = require('path');
  
  module.exports = function(source) {
    const configPath = path.resolve('config.json');
    
    // 添加文件依赖
    this.addDependency(configPath);
    
    // 读取配置文件
    const config = JSON.parse(fs.readFileSync(configPath, 'utf-8'));
    
    return applyConfig(source, config);
  };
  ```

- this.resourcePath

  获取当前处理文件的绝对路径

  ```js
  module.exports = function(source) {
    // 获取当前文件路径
    const filePath = this.resourcePath;
    // 添加文件头注释
    return `/* File: ${path.basename(filePath)} */\n${source}`;
  };
  ```

- this.resourceQuery

  获取资源URL中的查询参数

  ```js
  module.exports = function(source) {
    const query = new URLSearchParams(this.resourceQuery);
    if (query.has('minify')) {
      return minify(source);
    }
    return source;
  };
  ```

- this.emitError() & this.emitWarning()

  抛出错误和警告

  ```js
  module.exports = function(source) {
    if (source.includes('deprecatedFunction')) {
      this.emitWarning('Deprecated function used');
    }
    try {
      return transform(source);
    } catch (error) {
      this.emitError(error);
      return source; // 返回原始源码作为回退
    }
  };
  ```

### loader & plugin

- 区别
	- loader 
		loader是文件加载器，能够加载资源文件；
		并对这些文件进行一些处理，如：编译、压缩等；
		最终一起打包到指定的文件中。
		loader是一个转换器；
		将A文件进行编译形成B文件，这里操作的是文件；比如sass转成css；
	- plugin
		在webpack运行的生命周期中会广播出许多事件；
		plugin可以监听这些事件，在合适的时机通过webpack提供的api改变输出结果。
		plugin是一个扩展器，丰富webpack本身；
		针对是loader结束后webpack打包的整个过程；
		并不是直接操作文件，而是基于事件机制工作；
		会监听webpack打包过程中的某些节点，执行广泛的任务；
- loader
	- 原理：loader本质就是一个导出的函数，这个函数就是对加载内容处理的过程，这个函数需要接收一个参数，这个参数就是我们加载的文件，输出的就是我们加载后的结果。
	- 开发
		//markdown-loader.js
        module.exports=function(source){
            console.log(source)
            return 'hello loader'
        }
	- 执行顺序：从右往左
	- 常用
		- style-loader
			把样式插入到页面style标签里
		- css-loader
			处理css文件之间已依赖关系（js里引入css1，css1里引入css2）
		- postcss-loader
			处理css3等特性的浏览器兼容性
			（内部依赖autoprefixer插件，需要在plugins里对这个插件进行引用）
		- sass-loader
			处理sass
		- MiniCSSExtractPlugin.loader
			抽离css文件
		- babel-loader
			把ES6的代码转成ES5
			module: {
              rules: [
                {
                  test: /.js$/,
                  exclude: /node_modules/,
                  use: {
                    loader: 'babel-loader',
                    options: {
                      presets: ['@babel/preset-env'],
                      plugins: [
                      	[
                           '@babel/plugin-transform-runtime'
                     	]
                     ]
                    }
                  }
                }
              ]
            }
		- vue-loader
			解析和转换 .vue 文件；
			提取出其中的逻辑代码 script、样式代码 style、以及 HTML 模版 template；
			再分别把它们交给对应的 Loader 去处理。
		- file-loader
			处理图片
		- url-loader
			处理图片，根据options.limit 大小判断是否打包成base64格式；
			小图打包成base64，减少http请求；
			大图依旧像file-loader一样，单独打包到img文件夹里，发送请求，防止页面首次渲染太慢
        {
            test:/\.(png|jpg)$/,
                use: {
                loader: 'url-loader',
                options: {
                    limit: 5 * 1024，
                    outputPath: '/img/'
                }
            }
        }
- plugin
	- 原理：插件机制其实就是钩子；
	- 开发：
		- 这个plugin必须是一个函数或者一个包含apply方法的对象，一般都是定义一个类，在这个类中定义apply方法。
		- 在webpack启动时，这个类型生成一个对象，这个对象的apply方法，接收一个compiler参数，这个参数就是webpack工作最核心的对象，里面包括我们webpack构建过程中所有的配置信息，就是通过这个对象去注册相应的钩子函数；
		- emit钩子是生成资源到 output目录的时候调用，所以这个阶段最合适；
		- 传递过来的compiler对象的hooks属性，我们可以访问到这个emit钩子，再通过tap方法注册这个钩子函数，这个钩子函数接收两个参数，一个是插件的名，一个是要挂载到钩子上的函数，这个函数有一个参数叫做compilation，这个对象可以理解为打包过程中上下文，打包的所有结果都会放到这个对象里。
		  //remove-comments-plugin.js
        class RemoveCommentsPlugin{
            apply(compiler){
                compiler.hooks.emit.tap('RemoveCommentsPlugin',
                    function(compilation){
                    for(const name of compilation.assets){
                        console.log(name);
                    }
                })
            }
        }
        module.exports=RemoveCommentsPlugin;
	- 常用
### tree-shaking
- #### 原理

  - ##### 静态分析

    - 模块依赖分析
      - webpack构建过程中，会对项目中的模块进行依赖分析；解析每个模块的内容，确定模块之间的导入和导出关系
      - 通过分析，可以构建出一个模块依赖图，展示各个模块间的引用关系
    - 识别未使用的代码
      - 基于模块依赖图，确定哪些模块被实际引用，哪些模块未被使用
      - 对于 JavaScript 模块，它可以识别出未被调用的函数、未被访问的变量等。对于其他资源文件，如 CSS 和图片，也可以根据引用情况判断是否被使用

  - ##### 代码优化

    - 消除未使用的代码
      - webpack会在打包过程中将未使用的代码从最终的输出文件中移除
      - 可以显著减小打包文件的大小，提高应用的加载速度和性能
    - 作用域分析
      - 在消除代码时，还会进行作用域分析；确保在移除代码的过程中，不会影响到实际使用的代码正确性

  - ##### 实现条件

    - ES2015模块语法
      - Tree-Shaking主要依赖于ES2015模块语法（import、export）；这种模块语法是静态的，使得webpack能够在编译时确定模块的导入和导出关系
      - 相比之下，CommonJS模块语法（require、module.exports）是动态，难以在编译时进行准确的分析
    - 支持的模块类型
    - Webpack 不仅可以对 JavaScript 模块进行 Tree Shaking，还可以对一些其他类型的模块进行处理，如 CSS 模块（通过特定的加载器和插件）
    - 对于不同类型的模块，Webpack 可能会使用不同的技术和策略来实现 Tree Shaking

  - webpack5默认开启

- #### sideEffects

  - 如果所有代码都不包含副作用，可以直接设为false，告知可安全的删除未用到的export
```js
	{
      "name": "your-project",
      "sideEffects": false
    }
```
	- 有些代码存在副作用时，设置为数组

```js
    {
      "name": "your-project",
      "sideEffects": ["./src/some-side-effectful-file.js", "*.css"]
    }
```
- #### usedExports
```js
optimization: {
	usedExports: true
}
```
- #### sideEffects 更为有效 是因为它允许跳过整个模块/文件和整个文件子树。
- #### 前提
	
	- 使用es2015模块语法（即import，export）
	- 确保编译器没有把es2015模块转换成commonjs
### 通过 webpack 按需加载代码，提取第三库代码（splitChunk）
- 实现 提取第三方库 (webpack4的splitChunk)
	由于引入的第三方库一般都比较稳定，不会经常改变。所以将它们单独提取出来，作为长期缓存是一个更好的选择。
	这里需要使用 webpack4 的 splitChunk 插件 cacheGroups 选项。
```js
optimization: {
      runtimeChunk: {
        name: 'manifest' // 将webpack的runtime代码拆分为一个单独的 chunk。
    },
    splitChunks: {
        cacheGroups: {
            vendor: {
                name: 'chunk-vendors',
                test: /[\\/]node_modules[\\/]/,
                priority: -10,
                chunks: 'initial', // (all，async 和 initial)
            },
            common: {
                name: 'chunk-common',
                minChunks: 2,
                priority: -20,
                chunks: 'initial',
                reuseExistingChunk: true
            }
        },
    }
},
```
	test: 用于控制哪些模块被这个缓存组匹配到。原封不动传递出去的话，它默认会选择所有的模块。可以传递的值类型：RegExp、String和Function;
	priority：表示抽取权重，数字越大表示优先级越高。因为一个 module 可能会满足多个 cacheGroups 的条件，那么抽取到哪个就由权重最高的说了算；
	reuseExistingChunk：表示是否使用已有的 chunk，如果为 true 则表示如果当前的 chunk 包含的模块已经被抽取出去了，那么将不会重新生成新的。
	minChunks（默认是1）：在分割之前，这个代码块最小应该被引用的次数（译注：保证代码块复用性，默认配置的策略是不需要多次引用也可以被分割）
	chunks (默认是async) ：initial、async和all
	name(打包的chunks的名字)：字符串或者函数(函数可以根据条件自定义名字)
### webpack执行顺序
![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220212144013.png)
### 面试题
- 将图片插入到页面中
	- 解决方案
		module 下配置 rules
        module: {
            rules: [
                {
                    test:/\.(png|jpg)$/,
                    loader: 'file-loader'
                }
            ]
        }
- 抽离css
- 抽离公共代码

## babel
### 定义
	babel是一个js编译器，是一个工具链；
	主要用于将采用es6+语法编写的代码转化为向后兼容的js语法；
	以便能够运行当前和旧版本的浏览器或其他环境中；
### 本质
	本质就是在操作AST来完成代码的编译；
### 过程
	- 解析（Parse）（babel-parser）
		将源代码转换成抽象语法树；包括语法分析和词法分析；
		语法分析主要把字符流源代码转换成令牌流；
		语法分析主要将令牌流转换成抽象语法树；
	- 转换（Transform）
		通过babel插件能力，对AST做一些特殊处理；
		将高版本AST转换成支持低版本的AST；
		在此过程中也可以对AST的Node节点进行优化，比如添加、更新、移除等；
	- 生成（Generator）（babel-generator）
		将AST转换成字符串形式的低版本代码，同时也能创建SourceMap映射；
![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220209103500.png)
![](https://i-coder.oss-cn-beijing.aliyuncs.com/files/20220209103521.png)

### AST格式
https://astexplorer.net/
### babel
- babel-core：核心包，提供转译的 API，用于对代码进行转译。例如 babel.transform。在 webpack 中 babel-loader 就是通过这个包实现。
- babylon：babel 的词法解析器。将原始代码逐个字母地像扫描机一样读取分析得出 AST 语法树结构。
- babel-generator 将字符串代码转成抽象语法树 AST。
- babel-traverse：对 AST 进行遍历转译。
- babel-generator：将新的 AST 语法树生成新的代码。
- babel-types：用于检验、构建和改变 AST 树的节点
- babel-template：辅助函数，用于从字符串形式的代码来构建 AST 树节点
- babel-helpers：一系列预制的 babel-template 函数，用于提供给一些 plugins 使用
- babel-code-frames：用于生成错误信息，打印出错误点源代码帧以及指出出错位置
- babel-plugin-xxx：babel 转译过程中使用到的插件，其中 babel-plugin-transform-xxx 是 - - - transform 步骤使用的。
- babel-preset-xxx：transform阶段使用到的一系列的 plugin。
- babel-polyfill：JS 标准新增的原生对象和 API 的 shim，实现上仅仅是 core-js 和 regenerator-- runtime两个包的封装。
- babel-runtime：功能类似 babel-polyfill，一般用于 library 或 plugin 中，因为它不会污染全局作用域
### es6如何转成es5

> webpack本身是一个模块打包器（bundler），并不直接负责将ES6代码编译为ES5；webpack主要功能是将项目中的所有模块打包成一个或多个bundle，以供浏览器加载
>
> webpack可以通过loader和plugin扩展功能，实现代码的转换和编译

#### 步骤

- babel转换

- loader配置

- babel预设

  Babel 使用预设（presets）来定义转换规则。`@babel/preset-env` 是一个常用的预设，它会自动配置 Babel 以兼容目标浏览器的版本

- polyfills

  ​	为了支持旧浏览器，Babel 还可以引入 polyfills，这些是提供现代 JavaScript 特性的第三方代码片段。例如，`core-js` 和 `regenerator-runtime` 是两个常用的 polyfill 库

- 转换过程

  - 解析
  - 转换
  - 生成

- Tree Shaking

- 代码分割

- 优化

### webpack-dev-server

webpack-dev-server 是一个开发服务器，它提供了一个快速开发的环境，并且配合Webpack使用

- #### 作用

  - 自动编译和打包
    - 监听源文件的变化，当文件发生改动时，会自动重新编译和打包
  - 热模块替换
  - 代理和反向代理
    - 配置代理Proxy，解决开发环境跨域问题
  - 路由处理
  - 静态文件服务
