# 前端团队工程规范

> 本文档约定团队在所有前端项目中遵循的开发规范。
> **必须（MUST）** 表示强制要求；**应该（SHOULD）** 表示推荐做法；**可以（MAY）** 表示可选。

---

## 一、Git 工作流

### 1.1 分支策略

- **必须** 以 `main` 为唯一长期稳定分支，禁止直接向 `main` force push。
- 功能开发 **应该** 在短生命周期的特性分支上完成，命名格式 `feat/<简短描述>` 或 `fix/<简短描述>`，完成后删除。
- 小型项目或迭代节奏快的项目 **可以** 省略特性分支，直接推送 `main`，但须确保每次推送可编译。

### 1.2 Commit Message 规范（Conventional Commits）

格式：
```
type(scope): subject

[body]

[footer]
```

**type 取值**：

| type | 适用场景 | 影响 SemVer |
|------|---------|-------------|
| `feat` | 新功能、新页面、新接口 | MINOR ↑ |
| `fix` | Bug 修复 | PATCH ↑ |
| `refactor` | 重构（不改变外部行为） | PATCH ↑ |
| `perf` | 性能优化 | PATCH ↑ |
| `test` | 新增或修改测试 | — |
| `ci` | CI/CD 配置变更 | — |
| `chore` | 构建工具、依赖、配置更新 | — |
| `docs` | 文档变更 | — |
| `style` | 纯格式调整（不改逻辑） | — |

**规则**：
- **必须** 使用以上 type，不得自造 type。
- `scope` 为可选，填写受影响的功能模块（如 `auth`、`diagnosis`、`ui`）。
- `subject` **必须** 使用英文，首字母小写，不加句号，动词原形开头（`add`、`fix`、`remove`）。
- 破坏性变更 **必须** 在 footer 中注明 `BREAKING CHANGE: <描述>`，这会触发 MAJOR 版本号递增。

**示例**：
```
feat(diagnosis): add step-resume support for incomplete diagnoses
fix: remove duplicate error toast in response interceptor
chore: track package-lock.json for reproducible CI builds
ci: add build and release GitHub Actions workflows
```

---

## 二、版本控制规范

### 2.1 Semantic Versioning

所有项目 **必须** 遵循 [Semantic Versioning 2.0.0](https://semver.org/lang/zh-CN/)。

| 变更类型 | 版本递增 | 示例 |
|---------|---------|------|
| 破坏性 API / 架构变更 | MAJOR（X.0.0） | 1.x.x → 2.0.0 |
| 新功能，向后兼容 | MINOR（x.Y.0） | 1.1.x → 1.2.0 |
| Bug 修复、优化、测试、依赖更新 | PATCH（x.x.Z） | 1.1.0 → 1.1.1 |

### 2.2 Tag 与 package.json 同步

- Git tag **必须** 使用 `v` 前缀，格式 `vMAJOR.MINOR.PATCH`（如 `v1.2.3`）。
- **必须** 在发布 commit 中将 `package.json` 的 `version` 字段更新为与 tag 相同的版本号。
- `package.json` version 与 tag 不一致是 **错误**，CI 检测不到，需手动保证。

### 2.3 发布流程

```bash
# Step 1：在发布 commit 中更新 package.json version
#         并包含本次发布的所有功能/修复

git add <files> package.json
git commit -m "chore: release v1.2.3"

# Step 2：打 tag
git tag v1.2.3

# Step 3：推送 commit + tag（触发 CI Release Workflow）
git push && git push --tags
```

> Release Workflow 由 tag push 自动触发，无需手动操作 GitHub。

---

## 三、GitHub Actions 工作流模式

### 3.1 Build Workflow（持续集成）

**文件**：`.github/workflows/build.yml`
**触发**：向 `main` push 或提 PR

```yaml
name: Build

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```

**原则**：
- Node.js 版本 **应该** 固定为 LTS（当前 20），避免隐式升级。
- **必须** 使用 `npm ci`（严格依赖 lockfile），禁止 `npm install`。
- Build Workflow 只验证可编译性，不发布产物。

### 3.2 Release Workflow（持续交付）

**文件**：`.github/workflows/release.yml`
**触发**：push `v*.*.*` 格式的 tag

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write       # 必须：允许创建 GitHub Release

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # 必须：拉取完整历史，--generate-notes 依赖此项

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - name: Package dist
        run: zip -r dist-${{ github.ref_name }}.zip dist/

      - name: Create GitHub Release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create ${{ github.ref_name }} \
            --title "${{ github.ref_name }}" \
            --generate-notes \
            dist-${{ github.ref_name }}.zip
```

**原则**：
- `--generate-notes` 自动从相邻两个 tag 之间的 commit 生成 changelog，这是 Conventional Commits 规范的直接收益。
- 产物 zip 命名 `dist-{tag}.zip` 确保每次发布文件名唯一，方便回溯。
- `fetch-depth: 0` 与 `--generate-notes` **必须** 配套使用。

---

## 四、代码规范

### 4.1 格式化与 Lint

- **应该** 使用 **Biome** 统一替代 ESLint + Prettier，减少工具链依赖。
- Biome 配置 **必须** 纳入版本控制（`biome.json`）。
- 关键配置项：
  - Formatter：`indentStyle: space`
  - JS/TS 引号：`single`
  - 生成文件目录（`.umi/`、`dist/`、`node_modules/`）**必须** 加入 `files.includes` 排除列表。

### 4.2 Pre-commit Hook

- **必须** 配置 husky + lint-staged，在提交时自动格式化并检查暂存文件。
- `.lintstagedrc` 示例：
  ```json
  {
    "**/*.{js,jsx,tsx,ts,md,css,less,json}": [
      "npx @biomejs/biome check --write --no-errors-on-unmatched"
    ]
  }
  ```
- `--no-errors-on-unmatched`：避免仅提交被 Biome 排除的文件时报错退出。

### 4.3 依赖管理

- `package-lock.json` **必须** 纳入版本控制，确保 CI 与本地安装结果一致。
- 新增依赖前 **应该** 评估必要性（包大小、维护状态、是否有轻量替代）。
- CI 中 **必须** 使用 `npm ci` 而不是 `npm install`。

---

## 五、文件与命名规范（React/TypeScript 项目）

### 5.1 目录结构

```
src/
  pages/          # 路由页面，PascalCase 目录
    ModuleName/
      index.tsx       # 页面入口
      components/     # 页面专属组件
      hooks/          # 页面专属 Hook
      constants.ts    # 常量
  components/     # 全局共享组件
  services/       # API 调用层，kebab-case 目录
    module-name/
      index.ts        # 请求函数
      typings.d.ts    # 类型定义
  utils/          # 工具函数
  hooks/          # 全局共享 Hook
tests/            # 全局测试配置与 polyfill
```

### 5.2 文件命名

| 类型 | 规范 | 示例 |
|------|------|------|
| 页面/组件 | PascalCase，入口为 `index.tsx` | `UserDetail/index.tsx` |
| 子组件文件 | PascalCase.tsx | `EditPatientDrawer.tsx` |
| 自定义 Hook | `use` 前缀 + PascalCase | `useDiagnosisFlow.ts` |
| 服务目录 | kebab-case | `patient-user/` |
| 工具函数 | camelCase | `upload.ts`, `date.ts` |
| 测试文件 | 与被测文件同名 + `.test.ts` | `upload.test.ts` |
| 常量文件 | `constants.ts`，与使用方同目录 | `Diagnosis/constants.ts` |
| 类型文件 | `typings.d.ts` | `services/auth/typings.d.ts` |

### 5.3 代码命名

| 类型 | 规范 | 示例 |
|------|------|------|
| React 组件 | PascalCase | `EditPatientDrawer` |
| 自定义 Hook | `use` + PascalCase | `useDiagnosisFlow` |
| 普通函数/变量 | camelCase | `handleUploadChange` |
| 常量 | SCREAMING_SNAKE_CASE | `STATIC_PREFIX` |
| 布尔变量 | `is/has/can` 前缀 | `isLoading`, `hasError` |
| 类型/接口 | PascalCase | `DiagnosisDetailResponse` |
| API 字段名 | snake_case（与后端保持一致） | `chief_complaint` |
| 枚举 | PascalCase（值用 PascalCase 或 SCREAMING） | `DiagnosisStatus.Completed` |
