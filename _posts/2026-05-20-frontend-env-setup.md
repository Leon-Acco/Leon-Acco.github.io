---
layout: post
title: "前端开发环境搭建指南"
date: 2026-05-20
categories: [tools]
mermaid: false
math: false
toc: true
---
本文档详细记录前端开发环境的安装步骤、验证方法和常见问题排查，确保每一步都「装对了、能用了」。

---

## 一、Node.js 安装

Node.js 是前端开发的运行时基础，所有构建工具、包管理器、脚手架都依赖它。

### 方式一：nvm 安装（✅ 推荐）

nvm（Node Version Manager）可以在同一台机器上管理多个 Node.js 版本，随时切换。

**Linux / macOS：**

```bash
# 1. 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 2. 重新打开终端，或执行以下命令使 nvm 立即生效
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# 3. 安装最新 LTS 版本
nvm install --lts

# 4. 设置默认版本
nvm alias default node
```

**Windows：**

使用 [nvm-windows](https://github.com/coreybutler/nvm-windows/releases)，下载 `nvm-setup.exe` 安装即可。

```powershell
# 安装完成后，打开新的 PowerShell / CMD
nvm install lts
nvm use lts
```

### 方式二：官网直接安装

前往 [Node.js 官网](https://nodejs.org/) 下载 LTS 版本安装包，双击安装，一路默认即可。

### ✅ 验证 Node.js 安装成功

```bash
# 检查 node 版本（应输出 v20.x.x 或更高）
node -v

# 检查 npm 版本（随 node 一起安装）
npm -v

# 检查 nvm 版本（如果用 nvm 安装的话）
nvm -v
```

<div class="callout callout-success" markdown="1">
**安装成功的标志：**
- `node -v` 输出版本号，如 `v20.11.0`
- `npm -v` 输出版本号，如 `10.2.4`
- 执行 `node -e "console.log('Hello')"` 能正常输出 `Hello`

**常见问题：**
- `**node: command not found**` → 终端没有加载 nvm 配置，重新打开终端或检查 `~/.bashrc` / `~/.zshrc` 中是否有 nvm 初始化脚本
- **权限错误** → 不要用 `sudo` 安装 nvm，nvm 安装在用户目录下
- **Windows 路径问题** → 安装路径不要包含中文或空格
</div>

---

## 二、包管理器（pnpm）

npm 是 Node.js 自带的包管理器，但推荐使用 **pnpm**，速度更快、磁盘占用更少。

```bash
# 安装 pnpm（全局）
npm install -g pnpm
```

### ✅ 验证 pnpm 安装成功

```bash
pnpm -v
# 输出版本号，如 9.x.x
```

<div class="callout callout-tip" markdown="1">
**pnpm vs npm vs yarn 怎么选？**
- **pnpm**：速度最快，硬链接节省磁盘，严格依赖管理，推荐新项目使用
- **npm**：Node.js 自带，生态兼容性最好，不需要额外安装
- **yarn**：Facebook 出品，早期比 npm 快，现在优势不明显

选一个用就行，本文档以 **pnpm** 为主。
</div>

---

## 三、VS Code 配置

你大概率已经安装了 VS Code，这里主要关注**前端必装插件**。

### 必装插件清单

| **插件名称** | **用途** | **安装方式** |
| --- | --- | --- |
| ESLint | 代码规范检查，实时提示问题 | 扩展商店搜索 `ESLint` |
| Prettier | 代码自动格式化 | 扩展商店搜索 `Prettier` |
| Vue - Official（Volar） | Vue 3 语法高亮、类型提示 | 搜索 `Vue - Official` |
| Tailwind CSS IntelliSense | Tailwind 类名自动补全 | 搜索 `Tailwind CSS IntelliSense` |
| Auto Rename Tag | 自动同步修改 HTML 标签对 | 搜索 `Auto Rename Tag` |
| Error Lens | 行内直接显示错误信息 | 搜索 `Error Lens` |

### VS Code 推荐配置（settings.json）

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "emmet.includeLanguages": {
    "vue-html": "html"
  }
}
```

### ✅ 验证 VS Code 配置成功

<div class="callout callout-success" markdown="1">
- 打开一个 `.vue` 或 `.tsx` 文件，语法高亮正常
- 保存文件时自动格式化（缩进、引号等自动调整）
- 写一行明显的语法错误（如 `const a =`），ESLint 在编辑器中标红
</div>

---

## 四、Git 配置确认

你已经在用 Git，这里只需确认前端项目相关的配置。

```bash
# 确认 git 已安装
git -v

# 确认用户信息已配置
git config --global user.name
git config --global user.email
```

### ✅ 验证 Git 配置成功

<div class="callout callout-success" markdown="1">
- `git -v` 输出版本号
- `git config --global user.name` 输出你的名字
- `git config --global user.email` 输出你的邮箱
</div>

---

## 五、环境变量配置

前端项目中经常需要用环境变量来区分开发/生产环境、配置 API 地址等。这里分两部分：**系统级环境变量**和**项目级环境变量**。

### 5.1 系统级环境变量（PATH）

安装 Node.js、pnpm 等工具后，它们的可执行文件需要在系统 PATH 中才能在终端使用。如果你用 nvm 安装，通常会自动配置好。

**Linux / macOS：**

检查 `~/.bashrc`、`~/.zshrc` 或 `~/.bash_profile` 中是否包含以下内容：

```bash
# nvm 环境变量（安装 nvm 时自动添加）
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# pnpm 全局安装路径（如果 pnpm 命令找不到，手动添加）
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$PNPM_HOME:$PATH"
```

修改后执行 `source ~/.zshrc`（或 `~/.bashrc`）使其生效。

**Windows：**

1. 右键「此电脑」→「属性」→「高级系统设置」→「环境变量」
2. 在「用户变量」或「系统变量」中找到 `Path`，确认包含以下路径：

   ```javascript
   C:\Users\你的用户名\AppData\Roaming\nvm
   C:\Users\你的用户名\AppData\Roaming\nvm\nodejs
   C:\Users\你的用户名\AppData\Local\pnpm
   ```

3. 如果缺少，点击「新建」手动添加，然后**重新打开终端**

### ✅ 验证系统环境变量

```bash
# 查看当前 PATH 中是否包含 node、pnpm 路径
# Linux / macOS
echo $PATH
which node
which pnpm

# Windows PowerShell
$env:PATH
Get-Command node
Get-Command pnpm
```

<div class="callout callout-success" markdown="1">
**验证通过的标志：**
- `which node`（或 `Get-Command node`）能输出 node 的完整路径
- `which pnpm`（或 `Get-Command pnpm`）能输出 pnpm 的完整路径
- 在**任意目录**下都能执行 `node -v` 和 `pnpm -v`

**常见问题：**
- **新装的工具终端找不到** → 关闭并重新打开终端，环境变量才会生效
- **Mac 用了 zsh 但改的是 bashrc** → macOS 默认用 zsh，应该改 `~/.zshrc` 而不是 `~/.bashrc`
- **Windows 改了 PATH 不生效** → 必须重新打开 CMD/PowerShell，已打开的窗口不会自动刷新
- **多个 Node 版本冲突** → 用 `nvm list` 查看已安装版本，`nvm use <版本号>` 切换
</div>

### 5.2 项目级环境变量（.env 文件）

Vite 项目通过 `.env` 文件管理项目级环境变量，不同文件对应不同环境：

| **文件名** | **加载时机** | **用途** |
| --- | --- | --- |
| `.env` | 所有环境都加载 | 通用配置 |
| `.env.local` | 所有环境都加载，但被 git 忽略 | 本地私有配置（如密钥） |
| `.env.development` | `pnpm dev` 时加载 | 开发环境配置 |
| `.env.production` | `pnpm build` 时加载 | 生产环境配置 |

**示例**`**.env.development**`**：**

```bash
# Vite 项目中，变量名必须以 VITE_ 开头才能在前端代码中访问
VITE_API_BASE_URL=http://localhost:8080/api
VITE_APP_TITLE=My App (Dev)
```

**示例**`**.env.production**`**：**

```bash
VITE_API_BASE_URL=https://api.example.com
VITE_APP_TITLE=My App
```

**在代码中使用：**

```typescript
// 通过 import.meta.env 访问
const apiUrl = import.meta.env.VITE_API_BASE_URL
const appTitle = import.meta.env.VITE_APP_TITLE

console.log(apiUrl)  // 开发时输出 http://localhost:8080/api
```

**TypeScript 类型提示（可选但推荐）：**

在 `src/env.d.ts` 中声明类型：

```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_TITLE: string
  // 添加更多自定义环境变量...
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

### ✅ 验证项目环境变量

```bash
# 启动开发服务器
pnpm dev
```

<div class="callout callout-success" markdown="1">
**验证通过的标志：**
- 在组件中 `console.log(import.meta.env.VITE_API_BASE_URL)`，浏览器控制台能输出正确的值
- `pnpm dev` 和 `pnpm build` 分别读取到不同的 `.env` 配置
- 没有以 `VITE_` 开头的变量**不会**暴露到前端代码中（这是安全特性）

**常见问题：**
- **变量读取为**`**undefined**` → 变量名必须以 `VITE_` 开头，否则 Vite 不会注入
- **修改**`**.env**`**后不生效** → 需要重启 `pnpm dev`，`.env` 的修改不支持热更新
- `**.env.local**`**被提交到 Git** → 确认 `.gitignore` 中包含 `*.local`
- **生产环境泄露敏感信息** → 前端 `.env` 中**不要放密钥、密码**等敏感信息，它们会被打包到前端代码中
</div>

---

## 六、创建第一个项目（验证全链路）

这是最关键的一步——用脚手架创建一个项目并成功运行，说明环境全部就绪。

### 以 Vue 3 + Vite + TypeScript 为例

```bash
# 1. 创建项目
pnpm create vite@latest my-first-app -- --template vue-ts

# 2. 进入项目目录
cd my-first-app

# 3. 安装依赖
pnpm install

# 4. 启动开发服务器
pnpm dev
```

### 以 React + Vite + TypeScript 为例

```bash
pnpm create vite@latest my-first-app -- --template react-ts
cd my-first-app
pnpm install
pnpm dev
```

### ✅ 验证项目创建成功

<div class="callout callout-success" markdown="1">
**全链路验证通过的标志：**
4. `pnpm install` 没有报错，`node_modules` 目录已生成
5. `pnpm dev` 启动后，终端显示类似 `Local: http://localhost:5173/`
6. 浏览器打开该地址，能看到 Vite + Vue（或 React）的欢迎页
7. 修改 `src/App.vue`（或 `App.tsx`）中的文字，保存后**浏览器自动刷新**显示新内容

**常见问题：**
- `**pnpm: command not found**` → 重新执行 `npm install -g pnpm`，然后重新打开终端
- **端口被占用** → 终端会提示换端口，或手动指定 `pnpm dev --port 3000`
- **依赖安装慢** → 设置国内镜像：`pnpm config set registry https://registry.npmmirror.com`
</div>

---

## 七、项目工程化配置

项目能跑起来之后，还需要配置工程化工具保证代码质量。

### 7.1 ESLint + Prettier

```bash
# 安装 ESLint（Vue 项目）
pnpm add -D eslint @eslint/js typescript-eslint eslint-plugin-vue

# 安装 Prettier 及与 ESLint 的集成
pnpm add -D prettier eslint-config-prettier
```

创建 `eslint.config.js`：

```javascript
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import vue from 'eslint-plugin-vue'

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  ...vue.configs['flat/recommended'],
  {
    rules: {
      // 在这里自定义规则
    }
  }
]
```

创建 `.prettierrc`：

```json
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

### ✅ 验证 ESLint + Prettier 配置成功

```bash
# 运行 lint 检查
pnpm eslint src/

# 运行 prettier 格式化
pnpm prettier --check src/
```

<div class="callout callout-success" markdown="1">
- 命令能正常执行，不报「找不到配置」之类的错误
- 在 VS Code 中保存文件时自动格式化
- 写一段有问题的代码，ESLint 能在编辑器中标红提示
</div>

### 7.2 路由（以 Vue Router 为例）

```bash
pnpm add vue-router
```

### 7.3 HTTP 请求库

```bash
pnpm add axios
```

### 7.4 TypeScript 配置确认

Vite 脚手架已自动生成 `tsconfig.json`，确认以下关键项：

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

<div class="callout callout-success" markdown="1">
**TypeScript 验证：** 在 `.ts` 或 `.vue` 文件中写 `const a: number = 'hello'`，VS Code 应立即标红类型错误。
</div>

---

## 八、环境检查清单

最终确认所有环境都就绪：

- [ ] `node -v` → 输出 v20+
- [ ] `npm -v` → 输出版本号
- [ ] `pnpm -v` → 输出版本号
- [ ] `git -v` → 输出版本号
- [ ] 系统环境变量 PATH 正确（任意目录可执行 `node`、`pnpm`）
- [ ] VS Code 插件已安装（ESLint、Prettier、Vue/React 插件）
- [ ] 脚手架项目能 `pnpm dev` 成功启动
- [ ] 浏览器能访问 `localhost:5173` 看到欢迎页
- [ ] 修改代码后浏览器热更新生效
- [ ] 保存文件时自动格式化
- [ ] TypeScript 类型检查正常工作
- [ ] `.env` 环境变量文件已配置，代码中能正确读取
- [ ] npm 镜像已设置为国内源（可选但推荐）

<div class="callout callout-note" markdown="1">
**全部打勾 = 环境就绪，可以开始正式开发了！** 🚀
</div>
