## 1. 什么是 CI/CD？为什么需要？

**答案：**

- **CI（持续集成）**：代码频繁合并到主分支，并自动执行构建、测试，确保代码质量。
- **CD（持续交付/部署）**：在 CI 的基础上，将代码自动部署到测试/预发/生产环境。
- **好处**：
  - 减少人工操作错误
  - 代码质量可控
  - 发布流程标准化、自动化
  - 提升研发效率和交付速度

------

## 2. 前端项目常见的 CI/CD 流程是怎样的？

**答案：**

1. **代码提交到 GitLab/GitHub**
2. **触发 Webhook** → 通知 CI 工具（Jenkins/GitLab CI）
3. **CI/CD 流水线执行**
   - 安装依赖（npm install / pnpm install）
   - 代码检查（ESLint / Prettier / Stylelint）
   - 单元测试（Jest / Vitest）
   - 构建打包（Webpack / Vite / Rollup）
   - 生成构建产物（dist/）
4. **部署阶段**
   - 上传到测试环境 / 预发布 / 生产环境（SSH、Docker、K8s）
5. **通知**
   - 通过 **钉钉/企业微信/Slack** 发送构建结果
   - 通知开发人员或测试人员

------

## 3. GitLab Webhooks 在 CI/CD 中起什么作用？

**答案：**

- **作用**：当 GitLab 上发生 **push / merge request / tag / pipeline 事件** 时，通过 Webhook **回调通知 Jenkins 或其他服务**。
- **应用场景**：
  - Push 代码 → 自动触发 Jenkins 构建
  - Merge Request → 触发自动测试/预部署
  - Tag → 触发生产环境发布

------

## 4. Jenkins 在前端 CI/CD 中怎么用？

**答案：**

- **核心作用**：自动化执行流水线任务
- **常见配置**：
  - **Jenkinsfile** 编写 pipeline 脚本（分阶段执行：install → lint/test → build → deploy）
  - **GitLab Webhook** 触发 Jenkins job
  - **参数化构建**（选择环境：dev/test/prod）
- **部署方式**：
  - SSH 到服务器执行 `scp && pm2 restart`
  - Docker 构建镜像并推送到 Harbor / 阿里云镜像仓库
  - 调用 K8s 部署

------

## 5. 钉钉通知如何接入 Jenkins？

**答案：**

- **方式 1：Webhook 机器人**
  - 在钉钉群里创建自定义机器人，获取 Webhook URL
  - Jenkins job 执行完成后，调用 `curl` 或脚本 POST JSON 消息到钉钉
- **方式 2：插件**
  - 安装 `DingTalk Plugin`，在 Jenkins pipeline 里调用 `dingTalk()` API
- **示例：**

```
post {
  success {
    sh 'curl -H "Content-Type: application/json" -d \'{"msgtype": "text","text": {"content": "构建成功 ✅"}}\' https://oapi.dingtalk.com/robot/send?access_token=xxx'
  }
  failure {
    sh 'curl -H "Content-Type: application/json" -d \'{"msgtype": "text","text": {"content": "构建失败 ❌"}}\' https://oapi.dingtalk.com/robot/send?access_token=xxx'
  }
}
```

------

## 6. 前端项目中如何处理 PR（Merge Request）？

**答案：**

- **流程**：
  1. 开发在 feature 分支开发
  2. 提交 PR（Merge Request）
  3. GitLab CI/Jenkins 自动跑测试 + lint 检查
  4. 代码审核（Review）
  5. 通过后合并到 `develop` 或 `main`
- **好处**：
  - 保证代码质量（Lint、单测、E2E 测试）
  - 代码评审避免低级错误
  - 与 CI/CD 流水线集成，自动化验证

------

## 7. 你们的前端 CI/CD 流水线是如何分环境的？

**答案：**

- **常见做法：**
  - `develop` 分支 → 部署到测试环境
  - `release` 分支 → 部署到预发环境
  - `main/master` 分支 + tag → 部署到生产环境
- **部署策略**：
  - 通过 Jenkins 参数化选择部署环境
  - 通过 `.env.[env]` 配置环境变量（API 地址、静态资源地址等）

------

## 8. 如何保证前端构建产物的可追溯性？

**答案：**

- 在构建时写入 **版本号 / git commit hash / build time** 到页面或接口
- 产物命名加 hash（`app.[hash].js`）
- 通过 **Git Tag** 与部署版本对应

------

## 9. 前端 CI/CD 流程中常见的坑有哪些？如何解决？

**答案：**

- **依赖下载慢** → 使用 npm 私服 / cnpm / pnpm + 缓存
- **构建时间长** → 缓存 node_modules / 使用 Docker layer cache / 拆分子项目并行构建
- **环境变量错误** → 使用 dotenv / GitLab CI/CD 变量 / Jenkins credentials
- **前端回滚困难** → 保留历史构建产物，支持一键回滚
- **通知不准确** → 在 pipeline 里统一封装钉钉通知，保证失败及时推送

------

## 10. 如果面试官问：你如何设计一个前端 CI/CD 流程？

**参考答案：**

- 我会基于 **GitLab + Jenkins + Docker + 钉钉通知** 来设计：
  1. GitLab 代码提交 → Webhook 触发 Jenkins
  2. Jenkins pipeline：
     - 安装依赖
     - lint + 单测
     - 构建打包
     - 部署（SCP / Docker / K8s）
  3. 钉钉通知结果（成功/失败）
  4. PR 阶段跑自动测试，保证合并前的质量
  5. 分支策略保证环境隔离（dev/test/prod）
  6. 构建产物支持回滚 + 版本追溯