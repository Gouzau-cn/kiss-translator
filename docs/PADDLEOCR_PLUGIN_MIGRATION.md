# PaddleOCR 插件化迁移记录

> 日期：2026-11-16
> 范围：主仓库 `Gouzau-cn/kiss-translator` 与插件仓库 `Gouzau-cn/kiss-translator-plugins` 的 fork。
> 说明：本次工作仅操作上述 fork；上游仓库未保留本次新增的 PR（过程中创建过的上游插件 PR 已关闭）。

## 1. 目标与结论

上游维护者建议：新增服务接口统一走接口插件，主仓库只保留通用能力、调用流程和浏览器兼容处理。因此将 PaddleOCR 官方异步 API 的供应商适配从主仓库迁移到官方插件仓库，并修正 VLM 解析中的两个问题：

- 只读取 `markdown.text` 导致页眉、页脚、页码、脚注等文字丢失；
- 返回结果中的 `<div><img ...></div>` 图片占位符被当作正文送去翻译。

迁移后，主仓库不再包含 PaddleOCR 原生 provider；插件覆盖 PP-OCR 文字识别、PP-StructureV3 与 PaddleOCR-VL 系列文档解析，宿主仍使用原有的 `recognizeText` / `parseImage` 能力。

## 2. 主仓库改动

### 2.1 提交记录

| 提交 | 内容 |
| --- | --- |
| `f64dd7eb` | 重写 VLM 解析：优先 `parsing_res_list`，清理图片占位符，逐页合并版面块与 Markdown 回退页 |
| `d00f7b86` | 处理审查意见：Origin 守卫、坐标契约、混合页、无坐标块、模型名规范化 |
| `4a6470a8` | 移除原生 PaddleOCR provider、预设、配置卡片与测试 |
| `829f8a45` | 将上述改动合入 fork 的 `dev` 并推送 |
| `06b5bf21` | 文档：链接 PaddleOCR 接口插件（`docs/paddleocr-plugin` 分支） |

### 2.2 移除内容

- `src/builtin-plugins/paddleocr.js`、`paddleocrModels.js`、`paddleocr.test.js`
- `src/views/Options/PaddleOcrFields.js`
- `definitions.js` / `presets.js` / `fields.js` / `capabilities.js` / `runtime.js` 中的 PaddleOCR 分支
- `apiConstants.js` 中的 `OPT_TRANS_PADDLEOCR`
- 设置页中的 PaddleOCR 专用字段与对应多语言文案
- 相关测试断言与预设基线数量

### 2.3 保留内容

- 通用图像能力：`recognizeText`、`parseImage`、`analyzeLayout`
- `PADDLEOCR_ORIGIN` 与 `normalizeOriginRewriteList()`：PaddleOCR 网关会拒绝跨域 Origin，插件仍依赖这组 DNR 规则
- `updateCspRules()` 的 Origin 守卫：只修改 CSP 列表时不再清空用户已有的 Origin 规则（含回归测试 `src/background.csp.test.js`）

### 2.4 测试

- 移除后相关范围回归：104 个套件、1534 个用例全部通过。
- `dev` 分支全量测试：413 个套件通过、1 个跳过；7108 个用例通过、3 个跳过、0 失败。
- 预设基线恢复为移除 PaddleOCR 前的历史哈希与数量（35 个预设）。

## 3. 插件仓库改动

### 3.1 提交

`7bcafc1 feat: add PaddleOCR official async API plugin`

### 3.2 包结构

```text
plugins/paddleocr/
├── manifest.json
├── services/paddleocr.js
├── tests/paddleocr.test.cjs
├── assets/logo.png
├── README.md / README.en.md
└── ACCEPTANCE.md
```

### 3.3 服务与模型

| 服务 | 宿主能力 | 模型 |
| --- | --- | --- |
| 文字识别（PaddleOCR） | `recognizeText`，`coordinates: true` | PP-OCRv6、PP-OCRv5、PP-OCRv5-latin |
| 文档解析（PaddleOCR） | `parseImage`，`coordinates: false` | PP-StructureV3、PaddleOCR-VL、PaddleOCR-VL-1.5、PaddleOCR-VL-1.6 |

### 3.4 解析规则

- 逐页读取 `prunedResult.parsing_res_list`，保留 `markdown.text` 默认忽略的页眉、页脚、页码、脚注；
- 无版面块时回退到该页 `markdown.text`，混合页按页序合并；
- 移除 `<div><img></div>`、`<img>`、`![]()` 图片占位符；
- 表格 HTML 转纯文本时保留行列分隔并还原实体；
- 下载 BOS 预签名结果链接时不附加 Authorization；
- 文档解析固定 `useLayoutDetection: true` 且 `returnMarkdownImages: false`；
- 1.5/1.6 支持图片内文字识别（默认开启）与印章识别（默认关闭），旧版 VL 不发送。

### 3.5 测试与远程 CI

- 本地：`npm run check`、`npm test`、`npm run build` 全部通过；离线协议测试使用假 HTTP 响应，覆盖提交参数、轮询、JSONL 解析、图片占位清理、混合页、旧模型参数裁剪和错误路径。
- 远程：fork 内 PR #1 的 GitHub Actions 通过（check / test / build），合并到 fork `main` 后的发布工作流同样通过。
- `ACCEPTANCE.md` 明确记录：**尚未使用真实 AI Studio Token 调用厂家接口**，结果下载域名、重定向行为和浏览器 Worker/RPC 均未实测。

## 4. PR 与 CI 状态

| 仓库 | 分支/PR | 状态 |
| --- | --- | --- |
| fork 插件仓库 | PR #1（`feat/paddleocr-plugin → main`） | 已合并，远程 CI 通过 |
| fork 主仓库 | `dev` | 已更新并推送（`829f8a45`） |
| fork 主仓库 | PR #5（`docs/paddleocr-plugin → dev`） | 开启，远程 CI 暂未触发 |
| 上游仓库 | — | 本次未保留任何新增 PR，后续不对上游执行操作 |

主 fork 的 Actions 调度返回 `HTTP 422: Actions has been disabled for this repository`；`actions/permissions` 显示 `enabled: true`、工作流均为 `active`，切换开关后重试仍失败。需要确认仓库 Actions 状态后再触发 PR #5 的检查。

## 5. 已知限制与待办

1. **未实测真实接口**：需要 AI Studio Access Token 后才能验证各模型的识别质量、耗时与结果下载域名。
2. **结果下载域名**：插件默认允许 `https://paddleocr.bj.bcebos.com`，并提供“结果下载域名”配置项；宿主 HTTP 传输禁止重定向，预签名地址必须是直达链接。
3. **调用时长**：宿主插件调用默认 120 秒，插件内部任务预算 100 秒；长文档或复杂版面可能需要宿主提供可配置超时。
4. **任务模式**：文档解析当前仅声明 `doc`；表格/公式 `promptLabel` 模式需实测后再开放。
5. **宿主通用能力**：维护者建议的“单次解析包含版面分析（includesLayout）”与更通用的 Origin 策略仍属于后续独立改动。
6. **主 fork CI**：确认 Actions 可用后，重开 PR #5 或推送新提交触发检查，通过后合并。
