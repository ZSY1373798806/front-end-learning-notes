### rslib

### npm & pnpm & yarn

### **🔧核心机制差异**

| **特性**     | **npm**                            | **Yarn**                             | **pnpm**                                            |
| :----------- | :--------------------------------- | :----------------------------------- | :-------------------------------------------------- |
| **依赖管理** | 扁平化结构（可能重复依赖）         | 扁平化 + 确定性锁文件（`yarn.lock`） | **硬链接 + 符号链接**（全局存储复用）               |
| **锁文件**   | `package-lock.json`（JSON 格式）   | `yarn.lock`（紧凑文本格式）          | `pnpm-lock.yaml`（YAML + 哈希校验）                 |
| **依赖隔离** | ❌ 存在“幽灵依赖”（未声明包可访问） | ❌ 同 npm 的幽灵依赖问题              | ✅ **严格隔离**（仅访问显式声明的依赖）              |
| **存储方式** | 每个项目独立复制依赖               | 每个项目独立复制依赖                 | **全局存储**（`~/.pnpm-store`），项目通过硬链接引用 |

**pnpm 硬链接示例**：
若 100 个项目依赖相同 `lodash` 版本，磁盘仅存一份代码，项目通过硬链接复用，节省 60-80% 空间

### **⚡性能对比**

| **指标**               | **npm**        | **Yarn**           | **pnpm**                     |
| :--------------------- | :------------- | :----------------- | :--------------------------- |
| **安装速度**（无缓存） | 慢（20-30s）   | 快（8-12s）        | **极快**（3-9s）             |
| **缓存安装速度**       | 1-2s           | 4-5s               | **<1s**                      |
| **磁盘占用**           | 100%（基准）   | 90-100%            | **20-30%**                   |
| **Monorepo 支持**      | 基础（npm 7+） | 成熟（Workspaces） | **最优**（硬链接跨项目共享） |

> **实测场景**（1000 个依赖项目）：
>
> - `npm install`: 28.5s | 450 MB
> - `yarn install`: 12.8s | 420 MB
> - `pnpm install`: 6.3s | 180 MB

### 🛠️ **功能与生态对比**

| **能力**                    | npm                      | Yarn                 | pnpm                  |
| :-------------------------- | :----------------------- | :------------------- | :-------------------- |
| **Monorepo 命令**           | 需第三方工具（如 Lerna） | `yarn workspaces`    | `pnpm -r`（递归执行） |
| **安全审计**                | `npm audit`              | `yarn audit`         | `pnpm audit`          |
| **即插即用（PnP）**         | ❌                        | ✅（Yarn Berry 默认） | ❌                     |
| **零安装（Zero-Installs）** | ❌                        | ✅（Yarn Berry）      | ❌                     |
| **依赖补丁**                | ❌                        | `yarn patch`         | `pnpm patch`          |

### 组件调试

#### 方法1：使用 `pnpm link`

- 使用方式

  - 在包目录中创建全局链接

    ```bash
    cd D:\Data\CODE\npmjs\rslib-fetch
    pnpm link --global
    ```

  - 在项目目录中链接本地包

    ```bash
    cd D:\Data\CODE\my-project
    pnpm link --global rslib-fetch
    ```

  - 验证链接

    ```bash
    # 在项目目录中运行
    pnpm list rslib-fetch
    # 应该显示类似：
    # rslib-fetch link:D:\Data\CODE\npmjs\rslib-fetch
    ```

- 优点

  - 实时更新：包中的更改会立即反映在项目中
  - 无需重新安装
  - 支持热重载

#### 方法2：使用文件路径安装

- 使用方式

  ```bash
  cd D:\Data\CODE\my-project
  pnpm install D:\Data\CODE\npmjs\rslib-fetch
  ```

- 优点
  - 更接近真实安装环境
  - 不会污染全局命名空间

### changelog.md生成

- standard-version

  配置package.json

  ```json
  "scripts": {
    "release": "standard-version"
  }
  ```

  ```bash
  npm run release
  ```

- conventional-changelog

  ```json
  "scripts": {
    "changelog": "conventional-changelog -p angular -i CHANGELOG.md -s -r 0"
  }
  ```

  | 参数              | 说明                                                         |
  | :---------------- | :----------------------------------------------------------- |
  | `-p angular`      | 使用 **Angular 预设规范**（基于 [Conventional Commits](https://www.conventionalcommits.org/)），自动按 `feat`, `fix`, `break` 等类型分类提交记录。 |
  | `-i CHANGELOG.md` | 将生成的日志写入 `CHANGELOG.md` 文件（输入文件路径）。       |
  | `-s`              | 静默模式（不输出过程信息）。                                 |
  | `-r 0`            | 生成 **未发布版本（Unreleased）** 的变更日志（适用于尚未打 Git Tag 的新提交）。 |

  执行效果：

  1. **读取 Git 提交历史**：扫描项目的 Git 提交记录（需符合 Angular 规范）。
  2. **生成日志**：将提交按类型分组（如 `Features`, `Bug Fixes`, `Breaking Changes`）。
  3. **更新文件**：将生成的日志**插入到 `CHANGELOG.md` 文件顶部**（保留原有内容）。
  4. **未发布版本**：`-r 0` 会生成类似以下标题的日志：

### 单元测试

- vitest
- @vitest/coverage-v8

## npm

### package.json

### 📦 **一、基础标识属性**

| 属性              | 类型     | 必填 | 说明                       | 示例                                 |
| :---------------- | :------- | :--- | :------------------------- | :----------------------------------- |
| **`name`**        | string   | ✅    | 包名（npm 唯一标识）       | `"name": "my-package"`               |
| **`version`**     | string   | ✅    | 语义化版本号               | `"version": "1.0.0"`                 |
| **`description`** | string   |      | 包描述（npm 搜索展示）     | `"description": "A utility library"` |
| **`keywords`**    | string[] |      | 关键词数组（提升搜索排名） | `"keywords": ["react", "utils"]`     |
| **`license`**     | string   |      | 开源协议                   | `"license": "MIT"`                   |

### 🔗 **二、依赖管理**

| 属性                       | 类型     | 作用                                   | 示例                                               |
| :------------------------- | :------- | :------------------------------------- | :------------------------------------------------- |
| **`dependencies`**         | Object   | **生产环境依赖**（用户安装时自动下载） | `"dependencies": { "lodash": "^4.17.21" }`         |
| **`devDependencies`**      | Object   | **开发环境依赖**（不打包到生产）       | `"devDependencies": { "eslint": "^8.0.0" }`        |
| **`peerDependencies`**     | Object   | **同伴依赖**（宿主环境需安装的包）     | `"peerDependencies": { "react": ">=16.8" }`        |
| **`optionalDependencies`** | Object   | 可选依赖（安装失败不中断流程）         | `"optionalDependencies": { "fsevents": "^2.3.2" }` |
| **`bundledDependencies`**  | string[] | 打包进发布包的依赖（脱离 npm 分发）    | `"bundledDependencies": ["local-dep"]`             |
| **`overrides`**            | Object   | **强制指定依赖版本**（覆盖嵌套依赖）   | `"overrides": { "axios": "1.2.1" }`                |

### 🚀 **三、入口与文件配置**

| 属性          | 类型     | 作用                                | 示例                                  |
| :------------ | :------- | :---------------------------------- | :------------------------------------ |
| **`main`**    | string   | **CommonJS 入口文件**               | `"main": "./dist/index.js"`           |
| **`module`**  | string   | ES Module 入口（支持 Tree Shaking） | `"module": "./dist/index.mjs"`        |
| **`browser`** | string   | 浏览器专属入口                      | `"browser": "./browser.js"`           |
| **`types`**   | string   | TypeScript 类型声明文件             | `"types": "./dist/index.d.ts"`        |
| **`exports`** | Object   | **条件化导出**（Node.js 12+）       | [见下方详解]                          |
| **`files`**   | string[] | **发布到 npm 的文件白名单**         | `"files": ["dist", "types"]`          |
| **`bin`**     | Object   | **可执行命令映射**（CLI 工具）      | `"bin": { "my-cli": "./bin/cli.js" }` |

#### `exports` 高级用法：

```js
"exports": {
  ".": {
    "import": "./dist/index.mjs",   // ES Module
    "require": "./dist/index.cjs",  // CommonJS
    "types": "./dist/index.d.ts"    // TypeScript
  },
  "./feature": "./src/feature.js"   // 子路径导出
}
```

### ⚙️ **四、脚本控制**

| 属性                               | 类型   | 作用                | 示例                             |
| :--------------------------------- | :----- | :------------------ | :------------------------------- |
| **`scripts`**                      | Object | **自定义 npm 脚本** | `"scripts": { "build": "tsc" }`  |
| **`config`**                       | Object | 脚本环境变量注入    | `"config": { "port": 8080 }`     |
| **`preinstall`** **`postinstall`** | string | **生命周期钩子**    | `"postinstall": "node setup.js"` |

#### 完整生命周期钩子：

```bash
prepublish → prepare → prepublishOnly → prepack → postpack → publish → postpublish
preinstall → install → postinstall
preuninstall → uninstall → postuninstall
pretest → test → posttest
```

### 🌐 **五、发布配置**

| 属性                | 类型     | 作用               | 示例                                                         |
| :------------------ | :------- | :----------------- | :----------------------------------------------------------- |
| **`private`**       | boolean  | **禁止发布到 npm** | `"private": true`                                            |
| **`publishConfig`** | Object   | 发布时覆盖配置     | `"publishConfig": { "registry": "https://private.npm.org" }` |
| **`os`**            | string[] | 限定操作系统       | `"os": ["darwin", "linux"]`                                  |
| **`cpu`**           | string[] | 限定 CPU 架构      | `"cpu": ["x64", "arm64"]`                                    |
| **`engines`**       | Object   | **运行时环境要求** | `"engines": { "node": ">=18.0.0" }`                          |

### 📚 **六、文档与仓库**

| 属性               | 类型     | 作用             | 示例                                                         |
| :----------------- | :------- | :--------------- | :----------------------------------------------------------- |
| **`author`**       | Object   | 作者信息         | `"author": { "name": "Alice", "email": "alice@example.com" }` |
| **`contributors`** | Object[] | 贡献者列表       | `"contributors": [{ "name": "Bob" }]`                        |
| **`homepage`**     | string   | 项目主页         | `"homepage": "https://example.com"`                          |
| **`repository`**   | Object   | **代码仓库地址** | `"repository": { "type": "git", "url": "https://github.com/user/repo.git" }` |
| **`bugs`**         | Object   | 问题反馈地址     | `"bugs": { "url": "https://github.com/user/repo/issues" }`   |
| **`funding`**      | Object   | 赞助信息         | `"funding": { "type": "patreon", "url": "https://patreon.com/user" }` |

### ⚡ **七、现代优化属性**

| 属性               | 类型     | 作用                               | 示例                                        |
| :----------------- | :------- | :--------------------------------- | :------------------------------------------ |
| **`type`**         | string   | **模块类型** (`commonjs`/`module`) | `"type": "module"`                          |
| **`sideEffects`**  | boolean  | **Tree Shaking 标记**              | `"sideEffects": false`                      |
| **`workspaces`**   | string[] | **Monorepo 工作区配置**            | `"workspaces": ["packages/*"]`              |
| **`eslintConfig`** | Object   | 内联 ESLint 配置                   | `"eslintConfig": { "extends": "standard" }` |
| **`browserslist`** | string[] | 目标浏览器范围                     | `"browserslist": ["defaults"]`              |



