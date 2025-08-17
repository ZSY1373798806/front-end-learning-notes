### 自定义 Markdown 加载器实现

#### 创建 Markdown 加载器 (`markdown-loader.js`)

```js
// ./loaders/markdown-loader.js
const marked = require('marked');
const hljs = require('highlight.js');
const loaderUtils = require('loader-utils');
const { validate } = require('schema-utils');

// 定义 loader 选项的 JSON Schema
const schema = {
  type: 'object',
  properties: {
    highlight: { type: 'boolean' },
    breaks: { type: 'boolean' },
    gfm: { type: 'boolean' },
    sanitize: { type: 'boolean' },
    wrapperClass: { type: 'string' },
    renderer: { type: 'object' },
    headerPrefix: { type: 'string' }
  }
};

module.exports = function(source) {
  // 获取 loader 选项
  const options = loaderUtils.getOptions(this) || {};
  
  // 验证选项
  validate(schema, options, {
    name: 'Markdown Loader',
    baseDataPath: 'options'
  });
  
  // 标记为可缓存
  this.cacheable();
  
  // 添加文件依赖
  this.addDependency(this.resourcePath);
  
  // 配置 marked
  marked.setOptions({
    highlight: options.highlight !== false ? (code, lang) => {
      if (lang && hljs.getLanguage(lang)) {
        return hljs.highlight(lang, code).value;
      }
      return hljs.highlightAuto(code).value;
    } : null,
    breaks: options.breaks || false,
    gfm: options.gfm !== false,
    sanitize: options.sanitize || false,
    renderer: options.renderer ? createCustomRenderer(options.renderer) : null,
    headerPrefix: options.headerPrefix || ''
  });
  
  // 获取回调函数（支持异步）
  const callback = this.async();
  
  // 转换 Markdown
  marked.parse(source, (err, html) => {
    if (err) {
      callback(err);
      return;
    }
    
    // 生成 React 组件
    const wrapperClass = options.wrapperClass || 'markdown-body';
    const result = `
      import React from 'react';
      import PropTypes from 'prop-types';
      
      const MarkdownContent = ({ className, ...props }) => (
        <div 
          className={\`${wrapperClass} \${className || ''}\`}
          dangerouslySetInnerHTML={{ __html: \`${escapeHtml(html)}\` }}
          {...props}
        />
      );
      
      MarkdownContent.propTypes = {
        className: PropTypes.string
      };
      
      export default MarkdownContent;
    `;
    
    callback(null, result);
  });
};

// 创建自定义渲染器
function createCustomRenderer(customOptions) {
  const renderer = new marked.Renderer();
  
  // 自定义标题渲染
  if (customOptions.heading) {
    renderer.heading = (text, level) => {
      const escapedText = text.toLowerCase().replace(/[^\w]+/g, '-');
      return `
        <h${level} id="${escapedText}">
          <a href="#${escapedText}" class="anchor">${text}</a>
        </h${level}>
      `;
    };
  }
  
  // 自定义链接渲染
  if (customOptions.link) {
    renderer.link = (href, title, text) => {
      const isExternal = /^https?:\/\//.test(href);
      return `<a href="${href}" 
                 ${title ? `title="${title}"` : ''}
                 ${isExternal ? 'target="_blank" rel="noopener noreferrer"' : ''}>
                ${text}
              </a>`;
    };
  }
  
  return renderer;
}

// 转义 HTML 特殊字符
function escapeHtml(html) {
  return html
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}
```

#### 安装所需依赖

```bash
npm install marked highlight.js loader-utils schema-utils
```

#### 配置 Webpack

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  mode: 'development',
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: 'babel-loader'
      },
      {
        test: /\.md$/,
        use: [
          {
            loader: 'babel-loader',
            options: {
              presets: ['@babel/preset-react']
            }
          },
          {
            loader: path.resolve('./loaders/markdown-loader.js'),
            options: {
              highlight: true,
              gfm: true,
              breaks: false,
              wrapperClass: 'markdown-container',
              renderer: {
                heading: true,
                link: true
              }
            }
          }
        ]
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader']
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
      title: 'Markdown 加载器示例'
    })
  ],
  resolveLoader: {
    modules: [
      'node_modules',
      path.resolve(__dirname, 'loaders')
    ]
  },
  devServer: {
    static: {
      directory: path.join(__dirname, 'dist'),
    },
    compress: true,
    port: 9000,
    hot: true
  }
};
```

#### 使用

```react
import React from 'react';
import ExamplePost from './posts/example.md';
import './MarkdownViewer.css';

const MarkdownViewer = () => {
  return (
    <div className="markdown-viewer">
      <h1>Markdown 内容查看器</h1>
      <div className="content-container">
        <ExamplePost className="custom-markdown" />
      </div>
    </div>
  );
};

export default MarkdownViewer;
```

