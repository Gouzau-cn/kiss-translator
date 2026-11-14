# KISS Translator 项目结构说明

> 本文件是仓库的「结构索引」，用于快速定位代码与理解模块边界。
> 生成日期：2026-11-14；对应 `dev` 分支提交 `8aefe7ec`，版本 `2.2.3`。
> 目录、入口或构建目标发生变化时，请同步更新本文件。

## 1. 项目速览

| 项 | 内容 |
| --- | --- |
| 定位 | 简约、开源的双语对照翻译浏览器扩展 & 油猴脚本 |
| 版本 | `package.json` 的 `version`（2.2.3）；`extensionStoreVersion`（2.2.2）表示商店已发布版本 |
| 许可证 | GPL-3.0 |
| 包管理 | pnpm（工作流锁定 9.14.4） |
| UI 技术栈 | React 18 + MUI 5 + Emotion |
| 构建 | CRA 5 + `react-app-rewired` 自定义 webpack；本地 AI 单独用 Vite 6 |
| 测试 | CRA/Jest（主体）+ Vitest（`src/localAi`）+ `node --test`（构建断言） |
| 扩展规范 | Chrome/Edge 使用 Manifest V3；Firefox/Thunderbird 各自独立 manifest |
| 运行环境 | Node.js >= 20（推荐 22/24），pnpm >= 9 |
| 代码规模 | `src/` 约 1140 个文件，测试文件与实现文件基本一一对应 |

核心能力：整页双语翻译、划词/查词/悬停/输入框翻译、多服务对比的独立窗口、YouTube 双语字幕与 ASR、接口插件体系、端侧本地 AI（LLM/VLM/OCR/ASR/TTS/版面分析/JEV）、文件翻译（TXT/Markdown/PDF/EPUB/图片）、KISS-Worker / WebDAV / GitHub Gist 同步。

## 2. 我想改 X，该看哪里

| 需求 | 起点 |
| --- | --- |
| 整页翻译的 DOM 扫描、分段、译文渲染 | `src/libs/translator.js` |
| 翻译生命周期、快捷键、悬浮球、划词、输入框 | `src/libs/translatorManager.js` |
| 翻译请求发送、批量聚合、流式、上下文 | `src/apis/index.js` → `src/apis/trans.js` |
| 内置服务的参数与能力声明 | `src/builtin-plugins/definitions.js`、`runtime.js` |
| 外部插件安装、授权、调用 | `src/providers/` |
| 设置页 UI | `src/views/Options/` |
| 工具栏弹窗 | `src/views/Popup/` |
| 划词翻译框 / 对比面板 | `src/views/Selection/`、`src/components/TranslationPanel/` |
| 文件翻译页与流水线 | `src/views/FileTranslate/`、`src/fileTranslation/` |
| 图片翻译 | `src/views/ImageTranslate/`、`src/libs/imageTranslate/` |
| 本地 AI 界面与运行时 | `src/views/Options/LocalAiTabs/`、`src/localAi/` |
| YouTube 字幕 / ASR | `src/subtitle/` |
| 规则匹配、订阅、可视化编辑器 | `src/libs/rules.js`、`src/libs/subRules.js`、`src/views/RuleEditor/` |
| 存储、迁移、同步 | `src/libs/storage.js`、`src/libs/sync.js`、`src/libs/storageCoordination.js` |
| 后台消息、权限、CSP/Origin 改写 | `src/background.js`、`src/config/msg.js` |
| 构建入口与产物规则 | `config-overrides.js`、`src/scripts/build-task.mjs` |

## 3. 顶层目录

```
kiss-translator/
├─ .agents/skills/          # 仓库内技能（发布流程 release-kiss-translator）
├─ .github/
│  ├─ workflows/            # deploy-web.yml（手动部署 Web）、release.yml（tag 触发发布）
│  └─ ISSUE_TEMPLATE/
├─ docs/                    # 项目文档（提示词契约、文档索引）
├─ public/                  # 静态资源、HTML 模板、多语言 _locales、各平台 manifest
│  ├─ manifest.json         # MV3（Chrome/Edge）
│  ├─ manifest.firefox.json # Firefox
│  ├─ manifest.thunderbird.json
│  ├─ api/                  # 注入用静态脚本
│  ├─ content.html          # Web 版内容页模板
│  └─ file-translate.html   # 文件翻译页模板
├─ src/                     # 源码（见第 4 节）
├─ testdata/                # 字幕断句测试样本（随 Web 产物发布）
├─ config-overrides.js      # CRA webpack/jest/devServer 定制（三套构建配置）
├─ vite.config.local-ai.ts  # 本地 AI 运行时与 workers 的 Vite 构建
├─ package.json             # 版本、依赖、全部构建/测试脚本
├─ pnpm-lock.yaml / pnpm-workspace.yaml
├─ tsconfig.json            # TS 配置（allowJs，strict，排除 localAi 测试）
├─ .babelrc / .prettierrc / .prettierignore
├─ .env                     # 构建期 REACT_APP_* 变量（内容按安全规范不读取）
├─ README*.md               # 中文/英/日/韩说明
├─ CHANGELOG.md             # 发布变更记录（中文条目）
├─ VERSION_MANAGEMENT.md    # 版本管理规则
├─ custom-api.md / custom-api_v2.md  # 自定义 API 说明
└─ PROJECT_STRUCTURE.md     # 本文件
```

## 4. src 模块地图

| 目录 | 职责 | 关键文件 | 规模 |
| --- | --- | --- | --- |
| 根入口 | 各运行环境的启动入口与共享启动流程 | `background.js`、`content.js`、`common.js`、`options.js`、`popup.js`、`userscript.js`、`fileTranslate.js`、`fileBridge.js`、`injector-*.js`、`subtitleBootstrap.js` | 19 个顶层文件 |
| `apis/` | 翻译/词典/字幕/视觉请求调度；批量、缓存、流式、历史 | `index.js`、`trans.js`、`vision.js`、`history.js` | 26 |
| `builtin-plugins/` | 内置服务实现与能力契约（Google、DeepL、OpenAI、Claude、Gemini、百度、腾讯、火山、Yandex、Ollama、本地 AI 等） | `definitions.js`（能力矩阵）、`runtime.js`（注册）、`presets.js`、`fields.js`、`local.js` | 52 |
| `components/` | 跨页面复用 UI | `TranslationPanel/`、`FloatingButton`、`TerminologySelect`、`TouchTranslateControl` | 23 |
| `config/` | 常量与默认值：API 类型、默认设置、存储键、消息、提示词、多语言字典 | `index.js`（统一导出）、`api.js`、`apiConstants.js`、`setting.js`、`storage.js`、`msg.js`、`prompt.js`、`locales/` | 44 |
| `fileTranslation/` | 文件翻译协议、Web 端 client、扩展端 backend、图片流水线、结果缓存 | `protocol.js`、`client.js`、`backend.js`、`pipelineCore.js`、`resultCache.js` | 16 |
| `homepage/` | 官网/在线首页 App | `App.js`、`content.js`、`theme.js` | 6 |
| `hooks/` | React Context 与自定义 hooks（设置、主题、同步、字幕、弹窗等） | `Setting.js`、`Storage.js`、`Sync.js`、`Subtitle.js`、`Api.js` | 70 |
| `injectors/` | 注入到页面主世界/特定页面的脚本 | `shadowroot.js`、`xmlhttp.js`、`youtubeBridge.js`、`youtubeAudioBridge.js` | 7 |
| `libs/` | 核心业务逻辑（无 React 依赖为主） | `translator.js`、`translatorManager.js`、`storage.js`、`sync.js`、`rules.js`、`pool.js`、`batchQueue.js`、`document/`、`pdf/`、`epub/`、`diagnostics/` | 238 |
| `localAi/` | 端侧 AI：模型目录、运行时、Worker、推理缓存（TS 为主） | `offscreen.ts`、`core/catalog.ts`、`runtime/runtimeManager.ts`、`runtime/runtimePool.ts`、`workers/` | 122 |
| `providers/` | 插件运行时与 SDK：注册表、调用、安装、升级、声明式规则、平台适配 | `registry.js`、`invocation.js`、`package*.js`、`declarative.js`、`sdk/`、`contracts.ts` | 119 |
| `scripts/` | 构建、版本、打包、校验脚本（zx / node） | `build-task.mjs`、`local-ai-cache.mjs`、`update-version.mjs`、`sync-version.mjs`、`archive.mjs` | 29 |
| `styles/` | 全局样式与 M3 主题 | `m3.js`、`popupDocument.css` | 6 |
| `subtitle/` | YouTube 字幕抓取、AI 断句、ASR 全链路、字幕 UI | `subtitle.js`、`YouTubeCaptionProvider.js`、`BilingualSubtitleManager.js`、`asr/` | 89 |
| `views/` | 所有 React 页面 | `Options/`、`Popup/`、`Selection/`、`FileTranslate/`、`ImageTranslate/`、`RuleEditor/`、`Action/` | 273 |
| `testUtils/` | 测试辅助 | `scopedLogMock.js` | 1 |

> 命名约定：大多数实现文件旁边就是同名 `.test.js`；`localAi` 使用 `.test.ts` 且由 Vitest 运行。

## 5. 入口、构建与产物

### 5.1 三套 webpack 配置（`config-overrides.js`）

根据 `REACT_APP_CLIENT` 选择：

| 配置 | 触发值 | 用途 |
| --- | --- | --- |
| `extWebpack` | `chrome` / `edge` / `firefox` / `thunderbird` / `safari`（默认） | 浏览器扩展：popup、options、background、content、注入脚本 |
| `userscriptWebpack` | `userscript` | 油猴脚本产物（`build/web` 的 `.user.js`） |
| `webWebpack` | `web` | Web 站点/首页与在线文件翻译页；`REACT_APP_WEB_APP=homepage` 时只保留首页资源 |

### 5.2 入口一览

| 配置 | entry | 源码 | 产物 |
| --- | --- | --- | --- |
| ext | `popup` | `src/popup.js` | `popup.html` + `popup.js` |
| ext | `options` | `src/options.js` | `options.html` + `options.js` |
| ext | `background` | `src/background.js` | `background.js`（Service Worker） |
| ext | `content` | `src/content.js` | `content.js`（所有页面注入） |
| ext | `injector-subtitle` | `src/injector-subtitle.js` | YouTube 字幕注入 |
| ext | `injector-shadowroot` | `src/injector-shadowroot.js` | ShadowRoot 拦截注入 |
| ext | `file-bridge` | `src/fileBridge.js` | 文件翻译页 ↔ 扩展桥接 |
| ext | `subtitle-bootstrap` | `src/subtitleBootstrap.js` | 仅 Firefox 打包 |
| userscript | `main` | `src/index.js` | 在线首页 |
| userscript | `options` | `src/options.js` | 在线设置页 |
| userscript | `kiss-translator.user` | `src/userscript.js` | 油猴脚本 |
| web | `main` / `options` / `content` / `fileTranslate` | `src/index.js` / `src/options.js` / `src/userscript.js` / `src/fileTranslate.js` | `index.html`、`options.html`、`content.html`、`file-translate.html` |

### 5.3 构建目标矩阵（`src/scripts/build-task.mjs`）

| target | `REACT_APP_CLIENT` | 后处理要点 |
| --- | --- | --- |
| `chrome` | `chrome` | 删 Web 专属 HTML/manifest；注入 local-ai 运行时；向 manifest 追加 `file-bridge` content script |
| `edge` | — | 直接复制 `build/chrome` |
| `edge-dev` | `chrome` | 额外 `REACT_APP_FILE_BRIDGE_DEV=true`，同样注入运行时与 file-bridge |
| `firefox` | `firefox` | 替换为 `manifest.firefox.json`；注入运行时；追加 file-bridge；额外 `local-ai-background.html` |
| `thunderbird` | `thunderbird` | 替换为 `manifest.thunderbird.json` |
| `safari` | `safari` | 去掉 `clipboardRead` 可选权限，供 Svelte/Safari 工程使用 |
| `web` | `userscript` | 生成油猴脚本 + 调用 `homepage` 目标并合并首页资源；发布 `testdata` 字幕样本与 `version.txt` |
| `homepage` | `web` | 仅首页资源（`assets/homepage/*`），删除 manifest/_locales/api 等 |
| `edge` 构建前 | — | 必须先有 `build/chrome` |

产物目录：`build/<target>/`、`build/userscript/`（油猴汇总）、`build/web/`（GitHub Pages）；`.local-ai-build/` 为本地 AI 的 Vite 中间产物（均已在 `.gitignore` 中忽略）。

## 6. 核心运行链路

### 6.1 内容脚本启动（`content.js` → `common.js`）

1. 油猴模式先做 GM 兼容垫片与数据迁移；扩展模式由后台负责迁移。
2. 读取设置 → 初始化日志等级。
3. 非 HTML/文本/PDF 文档直接退出。
4. 黑名单校验（整页/划词/输入框/悬停可分别禁用）。
5. 空 iframe 跳过。
6. `matchRule()` 合并规则：**个人规则 > 订阅规则 > 全局规则**。
7. 构造 `TranslatorManager` 并 `start()`：整页翻译、划词、输入框、悬浮球、快捷键、跨 iframe 消息。
8. 顶级 frame 才启动字幕子模块；油猴模式还会触发订阅规则每日同步。

### 6.2 服务调用与插件体系

- 能力（capability）是选择服务的唯一依据：`translate`、`chat`、`summarize`、`detectLanguage`、`dictionary`、`suggest`、`segmentSubtitle`、`transcribe`、`recognizeText`、`parseImage`、`analyzeLayout`、`evaluate`、`synthesizeSpeech`、`speak`。
- 调用链：`config/serviceSelection.js` 按场景筛选可用服务 → `apis/index.js` 组装请求/缓存/批量 → `providers/registry.js` 统一 `invoke()` → 内置（`builtin-plugins/runtime.js`）或外部插件。
- 外部插件遵守 **Plugin API 1**：声明式规则支持 `format: "http" | "protocol"`；安装来源支持官方目录、ZIP、文件夹、GitHub 根地址、HTTPS；请求受声明的网络范围限制；实例与定义分离，支持升级与授权。
- SDK 位于 `src/providers/sdk/`（协议提示词 JSON、OpenAI 兼容封装、模板与流解析），与独立仓库 `kiss-translator-plugins` 配套。

### 6.3 本地 AI（端侧计算）

- 宿主：`offscreen.html`（`src/localAi/offscreen.ts`）运行 Runtime Host，通过 `BroadcastChannel`（RPC）响应 Service Worker 与页面请求。
- 组件：`RuntimeManager`（模型装载/推理）＋ `RuntimePool`（LRU、OOM 淘汰、pin 保护，JEV 最后卸载）＋ `TaskScheduler`。
- Worker：`asr`、`doc`、`jev`、`llm`、`llm-onnx`、`tts`、`ocr`（PaddleOCR），使用 ONNX Runtime Web / WebGPU，模型与中间文件存 OPFS/IndexedDB。
- 模型清单一律在 `src/localAi/core/catalog.ts`；`src/config/localModelTasks.json` 是任务测试页所需的轻量元数据（可由 `src/scripts/sync-local-model-tasks.mjs` 从目录生成）。
- 构建：Vite 输出到 `.local-ai-build`，由 `build-task.mjs` 经 `ensureLocalAi()` 复制进扩展；`pnpm build:chrome:clean` 或 `REBUILD_LOCAL_AI=1` 强制重建。
- webpack 侧将 `runtime/runtimeManager.ts` 别名到 `runtimeManager.webpack.js`。

### 6.4 文件翻译

1. 文件翻译页部署在 `src/config/fileTranslationSites.json` 列出的站点（默认 GitHub Pages），本地开发为 `localhost:3000/file-translate.html`。
2. 构建时为 chrome/edge-dev/firefox 的 manifest 追加 `file-bridge.js`，仅在上述页面 `document_start` 注入。
3. 页面（`fileTranslation/client.js`）通过 `window.postMessage`（`FILE_CHANNEL`）与 bridge 通信；bridge 用 `runtime.connect` 连到后台。
4. 后台 `installFileTranslationBackend()` 调用 `pipelineCore.js` / `libs/fileTranslationService.js` 完成解析、版面分析、OCR、翻译、导出（Markdown / HTML / ZIP / EPUB）。
5. 识别与翻译结果经 `resultCache.js` 缓存；图片链路按区域裁剪后走 OCR/VLM。

### 6.5 YouTube 字幕与 ASR

- 字幕获取：`injector-subtitle` + `injectors/youtubeBridge.js` 拦截页面字幕请求，`YouTubeCaptionProvider` / `youtubeCaptionTracks` 解析；`captionCache*` 做后台缓存。
- 无字幕视频：`subtitle/asr/` 提供 SABR 协议解析、AAC 解码、PCM 重采样、分片转录、任务与缓存管理。
- 断句与呈现：`sentenceBreaker.js`（规则）与 `youtubeAiSegmentation.js`（AI），`BilingualSubtitleManager` 管理双语字幕，`YouTubeSubtitleList` 提供对照列表；支持悬停查词、生词收藏与基于字幕的 AI 总结。

### 6.6 存储、规则与同步

- 本地键名带主版本号，例如 `<APP_NAME>_setting_v2`、`<APP_NAME>_rules_v2`；云端文件名如 `kiss-setting_v2.json`、`kiss-rules_v2.json`、`kiss-words.json`（见 `src/config/storage.js`）。
- 设置结构有独立版本号 `CURRENT_SETTINGS_VERSION`，迁移与升级前备份逻辑在 `src/libs/storage.js`。
- `src/libs/storageCoordination.js` 提供跨上下文写事务锁（IndexedDB 协调）。
- 同步后端：KISS-Worker、WebDAV、GitHub Gist（`src/libs/sync.js`，支持加密与冲突/回滚处理）。
- 规则来源：内置全局规则 `BUILTIN_RULES`（构建时由 `src/rules.js` 产出 `kiss-translator-rules.json`）、订阅规则、个人规则；可视化编辑器见 `views/RuleEditor/`。

## 7. 关键机制与约定

- **运行上下文标记**：各入口设置 `globalThis.__KISS_CONTEXT__`（`content` / `options` / `popup` / `fileTranslate`），共享库据此分支。
- **消息通信**：常量集中在 `src/config/msg.js`；后台在 `src/background.js` 的 `messageHandlers` 中分发；发送封装见 `src/libs/msg.js`。跨 iframe 用 `src/libs/iframe.js`。
- **环境适配层**：`src/config/client.js` 区分 web/chrome/edge/firefox/thunderbird/userscript；`src/libs/browser.js`、`gm.js`、`gmMethods.js` 抹平 `chrome.*`、`browser.*`、`GM_*` 差异。
- **提示词模板**：显式占位符与空模板规则见 `docs/prompt-template-contract.md`；默认模板在 `src/config/prompt.js` 与 `src/providers/sdk/*.json`。
- **术语库**：`src/libs/terminology*.js`（编译、匹配、应用、存储、同步、警告），场景术语与 AI 术语按 key 合并。
- **诊断日志**：`src/libs/diagnostics/` 负责记录、脱敏、导出、传输；设置页提供日志查看器。
- **安全**：不要在代码中写入明文密钥；服务凭据存于用户配置/同步数据中；`.env` 仅提供构建期常量。

## 8. 测试体系

| 命令 | 说明 |
| --- | --- |
| `pnpm test` | CRA/Jest，默认 `REACT_APP_CLIENT=chrome`；CI 下需 `CI=true` 单次运行 |
| `pnpm test:local-ai` | Vitest 运行 `src/localAi`（Jest 配置中显式忽略该目录） |
| `pnpm test:build` | `node --test` 校验打包边界与 chunk 结构 |
| `pnpm test:file-translation-build` | 校验文件翻译相关打包产物 |
| `pnpm test:subtitle-segmentation` | 字幕断句调试（babel-node） |
| `pnpm test:terms` | 术语替换调试（babel-node） |

Jest 特殊映射：`mp4box` 指向 CJS 版本，`googlevideo` 与 `@bufbuild/protobuf` 允许转译（见 `config-overrides.js` 的 `jest` 段）。

## 9. 常用命令

```sh
pnpm install            # 安装依赖
pnpm start              # Web 端开发调试（http://localhost:3000）
pnpm start:userscript   # 油猴脚本开发调试
pnpm build:chrome       # Chrome/Edge 扩展 → build/chrome
pnpm build:firefox      # Firefox 扩展 → build/firefox
pnpm build:web          # Web + 油猴脚本 + 首页 → build/web
pnpm build:local-ai     # 单独重建本地 AI 运行时
pnpm build:rules        # 生成内置规则 JSON
pnpm build              # 全量构建所有平台
pnpm build+zip          # 全量构建并打包发布物
pnpm format             # Prettier 格式化
pnpm check:pr           # PR 前预检：格式化 + Web/规则/Chrome 构建
pnpm sync-version       # 将 package.json 版本同步到各 manifest/.env
pnpm version:patch|minor|major|set -- <ver>  # 版本升级
```

## 10. 开发约定

- 编码统一 **UTF-8**，源码注释使用中文。
- Prettier：双引号、分号、80 列、2 空格缩进、`trailingComma: es5`、`endOfLine: auto`。
- CRA 生产构建把警告视为错误；`check:pr` 会依次构建 Web（含首页）、iOS 油猴脚本、规则、Chrome。
- 测试与实现同目录放置，命名 `<module>.test.js`（localAi 为 `.test.ts`）。
- 修改共享库时注意四个运行上下文（content/options/popup/fileTranslate）与三种存储后端（extension / GM / localStorage）。
- `build/`、`.local-ai-build/`、`node_modules/`、日志与压缩包均不提交。

## 11. 发布流程

1. `dev` 分支执行对应 `pnpm version:*`，并在 `CHANGELOG.md` 顶部新增 `## v<版本>` 中文条目。
2. `pnpm build+zip` 自检产物版本号一致。
3. 提交 `chore: bump version to <version>` 并推送 `dev`。
4. 创建 `dev → master` 的 `Release v<version>` PR，检查通过后合并。
5. 在 `master` 打注解 tag `v<version>` 并推送，触发 `.github/workflows/release.yml` 构建、部署 Web、创建 GitHub Release。
6. 将 `master` 快进同步回 `dev`。

完整约束与自动化步骤见 `.agents/skills/release-kiss-translator/SKILL.md` 和 `VERSION_MANAGEMENT.md`。

## 12. 常见改动指引

| 改动 | 需要同步的位置 |
| --- | --- |
| 新增内置翻译/其他服务 | `config/apiConstants.js`（类型常量与 `OPT_ALL_TRANS_TYPES`）→ `builtin-plugins/definitions.js`（能力契约）→ `builtin-plugins/fields.js`（设置字段）→ `builtin-plugins/runtime.js`（注册 handler）→ 具体实现文件 → `builtin-plugins/presets.js`（默认参数与是否默认启用）→ 补测试 |
| 新增外部插件能力 | `providers/contracts.ts`（输入/输出契约）→ `providers/registry.js`/`invocation.js`（校验与调用）→ `providers/sdk/`（公共封装）→ 兼容性测试 |
| 新增后台消息 | `config/msg.js` 定义常量 → `background.js` 的 `messageHandlers` 处理 → `libs/msg.js` 提供发送封装 → 补测试 |
| 新增 React 页面 | `views/<Name>/` → 新建入口 `src/<name>.js` → `public/<name>.html` → 在 `config-overrides.js` 增加 entry 与 HtmlWebpackPlugin → `build-task.mjs` 调整清理/复制规则 |
| 新增本地模型 | `localAi/core/catalog.ts`（清单与哈希）→ 对应 worker/推理封装 → 运行 `sync-local-model-tasks` 更新 `config/localModelTasks.json` → 设置页模型选择与测试页 |
| 新增存储字段 | `config/setting.js`（默认值）→ `CURRENT_SETTINGS_VERSION` 与 `libs/storage.js` 迁移 → 同步序列化/回归测试 |

## 附：相关文档

- [README.md](README.md)：功能、安装与开发指引
- [VERSION_MANAGEMENT.md](VERSION_MANAGEMENT.md)：版本号与发布规则
- [CHANGELOG.md](CHANGELOG.md)：历史变更
- [docs/README.md](docs/README.md)：文档索引
- [docs/prompt-template-contract.md](docs/prompt-template-contract.md)：提示词模板执行规则
- [custom-api.md](custom-api.md) / [custom-api_v2.md](custom-api_v2.md)：自定义 API 说明
