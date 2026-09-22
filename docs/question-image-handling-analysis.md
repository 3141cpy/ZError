# ZError 题干/选项中图片与非文本数据处理机制 — 深度分析报告

> 项目：ZError —— 配合 OCS 网课助手的本地 AI 题库软件（Tauri 2 + Vue 3 + Rust + SQLite）
> 分析范围：题干（question）、选项（options）、答案（answer）中存在图片（图片 URL）或其他非文本数据时的完整处理链路
> 报告中所有引用格式为 `文件路径 (行号)`，可对照源码阅读

---

## 1. 核心结论（一句话版）

**ZError 从不把图片当作二进制数据存储或传输——图片一律以 `http(s)://` URL 文本的形式内嵌在题目文本字段中**；只有在两个真正需要"像素"的消费端（① 本地 UI 渲染 ② 视觉大模型答题）才通过 Rust 后端把 URL **按需拉取并物化为 base64 data URL**，且物化结果带磁盘缓存。整条链路围绕"**文本即存储、URL 即图片、按需物化、多处兜底**"展开。

---

## 2. 总体数据流

```
学习平台页面 ──(题目文本, 图片以 URL 形式内嵌)──▶ OCS 网课助手
                                                    │ POST /api/questions/query (title, options, type)
                                                    ▼
                                   ┌─────────────────────────────────┐
                                   │ Rust 本地 HTTP 服务  server.rs    │
                                   │ 1. contains_url 检测 has_url      │
                                   │ 2. 查题库(精确/模糊+同题判断)      │
                                   │ 3. 未命中 → __URL_QUESTION__:    │
                                   │    <title>\n__OPTIONS__:<options> │
                                   │    通过 SSE/Tauri 事件发给前端     │
                                   │ 4. wait_for_model_response 阻塞  │
                                   └─────────────────────────────────┘
                                                    │ model-call-request 事件
                                                    ▼
                                   ┌─────────────────────────────────┐
                                   │ Vue 前端  Home.vue               │
                                   │ dispatchModelCallRequest 按前缀   │
                                   │ 路由到 handleUrlQuestionRequest  │
                                   │  → analyzeUrlQuestion (视觉分析)  │
                                   │   · 拆分文本/图片 URL 交错片段     │
                                   │   · Rust 抓图→base64(带磁盘缓存)  │
                                   │   · canvas 预处理(白底/放大)      │
                                   │   · 组装多模态消息发给 VLM         │
                                   │   · 流式展示 + 解析 ANSWER:       │
                                   └─────────────────────────────────┘
                                                    │ sendModelResponseToBackend
                                                    ▼
                                   后端收到答案 → 规范化 → 入库(SQLite, 仍存 URL 文本) → 返回 OCS
```

同时，题库管理页面（列表/详情/编辑）对含图题目的渲染、以及导入导出对图片的处理，都建立在这套"URL 文本"模型之上，详见第 6/7/8 节。

---

## 3. 数据模型层：图片就是文本

### 3.1 SQLite 存储结构

- 表 `AIResponses`：`Question / Options / Answer / QuestionType` 全部为 TEXT 列（见 `src-tauri/src/database.rs (1589-1616)` 的 `insert_ai_response`）。
- **没有 BLOB 列、没有图片表、没有附件机制**。一张"看图题"在库里就是一段含 `https://...png` 字符串的普通文本。
- 前端类型定义同构：`AIResponse { question: string; options?: string; answer?: string; ... }`（`src/services/database.ts (11-22)`）。

### 3.2 这个设计的得失

| 优点 | 代价与对策 |
|---|---|
| 存储极简，题库文件可随 SQLite 整体备份/导出 | 图片依赖源站存活 → Rust 端做**磁盘缓存**（见 5.2） |
| 检索/模糊匹配/导出全部复用纯文本逻辑 | UI 与 VLM 需要像素 → **运行时物化**（base64） |
| 导入导出天然兼容 CSV/Excel/Word/PDF | 编辑图片题只能改 URL 文本（编辑器是纯 textarea） |

> 其他非文本数据（音频/视频/附件）：项目中**不存在**相关处理代码。即本软件对非文本数据的边界就是"图片 URL"，再无其他形态。

---

## 4. 图片 URL 识别层：本地算法 + 云控热更新（`src/utils/questionImage.ts`）

这是整个图片处理体系的"地基"——所有下游（展示、拆分、VLM 组装）都依赖它判断"哪里是图"。

### 4.1 三个核心 API（对外统一出口，L425-438）

```ts
findQuestionImageMatches(text) : QuestionImageMatch[]   // 精确到 start/end 偏移的 URL 匹配
extractQuestionImageUrls(text) : string[]               // 去重后的 URL 列表
splitQuestionImageParts(text) : QuestionImagePart[]      // 文本/图片 交错的语义分段
// QuestionImagePart = { type: 'text' | 'image', text?, url? }  (L4-8)
```

`QuestionImageMatch`（L10-16）额外携带 `rawUrl / normalizedUrl / start / end / trailingText`，`trailingText` 保留 URL 末尾粘连的标点（如 `。png。` 后面的句号），保证拆分后文本不丢字。

### 4.2 本地默认算法（L44-66）

一个 RFC3986 风格的正则匹配 `http(s)://` URL，`normalizeUrl` 负责去掉末尾 `.,;!?` 等标点。**它不判断扩展名**——文本里出现的任何 http(s) URL 都按"疑似图片"处理（这是宁滥勿缺的策略，因为学习平台图片 URL 常常没有扩展名，如带 token 的 CDN 链接）。

### 4.3 云控热更新算法（本项目最有特色的设计，L252-382）

图片识别算法可以由**远程目录服务器下发 JS 代码动态替换**，无需发版：

1. `fetchRemoteModelsCatalog()`（`src/services/modelConfig.ts (339-360)`，带冷却时间缓存）拉取远程目录，字段 `questionImageAlgorithm`（`modelConfig.ts (100, 160-166)`，兼容 `question_image_algorithm` 等多种命名）。
2. `compileQuestionImageAlgorithm`（L320-342）用 `new Function('module','exports','runtime', ...)` 编译代码字符串：
   - `{...}` / `(...)` 开头的代码包装为 `module.exports = (...)`，也支持 `createQuestionImageAlgorithm(runtime)` 工厂函数与 `export default` 前缀剥离；
   - 编译产出必须通过 **preview 校验**（L290-297）：用样例文本 `'题目 https://example.com/demo.png) 结束'` 试跑，匹配不出任何 URL 则拒绝启用；
3. 每个导出函数都被再包一层**归一化+校验**（L298-318）：`start/end` 必须是合法且不越界不重叠的整数偏移（`normalizeQuestionImageMatches`, L148-166 做排序去重叠），parts 必须能兜底回原文（`normalizeQuestionImageParts`, L98-108）。
4. 失败路径完备：远程拉取失败/编译失败/运行时抛错 → **逐级回退本地算法**（L384-423 每个 ByAlgorithm 函数都有 try/catch + fallback），远程请求有 30 秒重试冷却（L252）。

**为什么需要云控**：各学习平台（超星/智慧树等）的题目 DOM→文本提取规则会变化，URL 可能被换行截断、带特殊转义等；算法云端可更新意味着识别规则可以随时热修。这是"数据驱动的策略与代码分离"的教科书案例。

---

## 5. 图片物化层：Rust 抓取 + 磁盘缓存（`src-tauri/src/commands.rs`）

### 5.1 `fetch_image_as_base64` Tauri Command（L92-173）

前端无法直接 `fetch` 平台图片（CORS + 防盗链），因此物化动作全部走 Rust：

- **磁盘缓存优先**：`{app_local_data_dir}/image_cache/{hash(url)}.b64`，命中直接返回（L98-112）。这一层缓存让"源站图片已失效的老题"仍能在本地渲染和重新喂给 VLM——相当于给题库的图片做了**离线副本**（存的是 data URL 文本而非图片二进制）。
- **三级反防盗链请求策略**依次尝试（L117-172）：
  1. **完整浏览器伪装**（L175-221）：Chrome UA + 完整 `Sec-Fetch-*` 头 + Cookie + 10 跳重定向，且 **Referer 按域名定制**——`chaoxing.com → https://mooc1-1.chaoxing.com/`、`zhihuishu.com → https://www.zhihuishu.com/`、其余回退 Google（针对两大网课平台的 Referer 白名单防盗链逐一适配）；
  2. **简化请求头**（L224-235）；
  3. **移动端 UA 伪装**（L238-250）。
- **魔数嗅探 Content-Type**（`detect_image_type`, L253-294）：按 PNG/JPEG/GIF/WebP/BMP 文件头判定，不信任服务器 Content-Type，拼成 `data:{mime};base64,...` data URL 返回。
- 全部失败返回 `Err`，前端 `fetchQuestionImageBase64`（`questionImage.ts (448-458)`）catch 后返回 `''`，由 UI 显示 `[图片: url]` 占位、由 VLM 流程直接报错（见 7.4）。

---

## 6. 答题主链路：URL 题从 OCS 请求到 VLM 应答（核心流程）

### 6.1 后端检测与路由（`src-tauri/src/server.rs`）

1. **URL 检测**：`contains_url`（L89-93，正则 `https?://[^\s]+`）对 `title` 和 `options` 分别检测，得出 `has_url`（L477-482）。
2. **先查题库**（URL 题也可能已有答案）：
   - `query_database_exact` → `score_question_row`（`database.rs (1370-1408)`）：**若查询题干含 URL，则候选题目的 URL 列表必须与查询完全一致**（L1381-1387，`extract_urls` 提取排序后比较）——防止"文字相同但配图不同"的两道题互相误匹配；
   - 相似度计算 `compute_query_match_score`（`database.rs (221-250)`）中，`normalize_urls`（L35-39）先把双方文本里的 URL **替换为 `__URL__` 占位符**再算 Levenshtein + jieba 关键词覆盖率——防止 URL 差异淹没正文相似度。
   - 一句话：**URL 在"精确匹配门禁"里是强约束，在"文本相似度"里是弱化噪声**，两种角色分开处理，非常巧妙。
3. 模糊候选 → `__SAME_QUESTION_CHECK__:` AI 同题判断（L519-649），此处不展开。
4. 未命中 → `wait_and_store_ai_answer`（L378-468）：
   - `build_normal_model_query`（L346-362）：**has_url 时不用常规答题提示词模板**，而是构造 `__URL_QUESTION__:<title>\n__OPTIONS__:<options>` 协议串（因为图片题必须交给前端视觉流程，文字模板没有意义）；
   - 通过 `logger.send_model_call_request`（SSE 日志通道）+ `emit_model_call_request`（Tauri 事件，L364-376）**双通道**推给前端；
   - **等待预算按是否含图区分**（`model_wait_budget_secs`, L45-63）：`absolute = timeout × (retries+1) + 90`，URL 题比普通题多 90 秒硬上限（下载图片 + VLM 推理更慢）；
   - `wait_for_model_response` 阻塞等待前端回传最终答案 → `extract_answer_from_json` → `normalize_answer_against_options` → `insert_ai_response` 入库（题干**原文**连同 URL 文本一起入库）。

### 6.2 前端路由（`src/views/Home.vue`）

- `dispatchModelCallRequest`（L2560-2585）按前缀判 phase：`__URL_QUESTION__:` → `'url'`；`__SAME_QUESTION_CHECK__:` → `'same'`；否则 `'answer'`。以 `requestId:phase` 为 key 去重（L2551），兼容 SSE/Tauri 双通道重复投递。
- `'url'` phase → `handleUrlQuestionRequest`（L3440-3540）：
  - 从协议串中剥出 `title` 与 `__OPTIONS__:` 内嵌的选项（L3447-3452）；
  - 在请求日志上挂 `urlQuestion` 状态对象（标题/选项/分析结果/流式内容等，L631-642）；
  - **立即并行做两件事**：`buildRenderedHtml`（L3980-3999，把 URL 全部换成 base64 `<img>`，供请求详情面板 `v-html` 展示）+ `analyzeUrlQuestion`（真正的 VLM 答题）。

### 6.3 视觉分析 `analyzeUrlQuestion`（L4086-4281）

1. **题型分类** `classifyUrlQuestionMode`（`src/utils/urlQuestion.ts (106-147)`）：根据 `type` 字段中英关键词判 `single/multiple/judgement/open`；type 为空时**只有解析出 ≥2 个带标签选项才视为单选**，否则按简答处理（防误判）。
2. **选项解析** `parseUrlOptions`（L84-104）：优先识别 `A.`/`1.` 标签块（**支持块内换行**，L7-67 的状态机逐行聚合，字母标签优先于数字标签）；仅当题型明确为选择/判断时才允许"无标签按非空行拆分"兜底。
3. **提示词构建** `buildUrlQuestionPromptParts`（L252-284）：
   - 选择题：选项重编号为 `1. 2. 3.` 文本列表；
   - open 题：原选项文本作为"【补充材料】"**不编号**（明确提示模型"不要把补充材料当成选项列表"）；
   - `answerRule` 按题型生成强约束，统一要求**末行单独输出 `ANSWER: <答案>`**。
4. **多模态组装** `buildMultimodalContent`（`Home.vue (4052-4082)`）——全链路最关键的一步：
   ```
   fullText = title + optionsText + instruction
     → findQuestionImageMatches 拆出所有 URL 位置
     → buildQuestionImageMap（L3958-3977）：每个 URL
         fetchQuestionImageBase64（Rust 抓图+缓存）
         → applyWhiteBackgroundToDataUrl（透明图合成白底）
         → ensureDataUrlMinimumSize(32)（最小 32px）
     → 文本/图片交错输出 [{type:'text'},{type:'image_url',image_url:{url,detail:'high'}}...]
   ```
   **任一图片下载失败即整体 throw**（L4077-4079）——宁可报错也不给 VLM 一道"看不全图"的题，避免模型瞎猜出错误答案入库存脏数据。
5. **发送前预处理** `prepareVisionRequestContent`（L3918-3927）再统一做一次最小尺寸放大。
6. **协议适配执行**：`resolveExecutableModelJsCode`（`modelProtocol.ts (717-743)`）按模型的 `apiProtocol` 生成/取回可执行 JS 代码（见第 8 节），通过 Tauri HTTP 插件 fetch（绕浏览器 CORS）调用，全程流式。
7. **失败自动放大重试** `executeVisionModelWithAutoUpscale`（L3930-3956）：捕获"图片尺寸不足"类报错（`extractVisionImageSizeError`, L3877-3891，识别错误码 `20015` 及 `height(N) or width(N)` 文本，取 max(N)+16 并 clamp 到 32~96px），canvas 放大后**自动重试一次**——针对部分 VLM（如 Qwen-VL 系）拒绝小图的兼容性工程。
8. **流式与心跳**：`streamingResponse/streamingReasoning` 增量更新 UI；每 1 秒 `sendModelProgressToBackend` 心跳（L4190-4197），维持后端等待时钟防 408。
9. **答案解析** `resolveUrlAnswer`（`urlQuestion.ts (178-250)`）：
   - `extractAnswerRaw` 取**最后一次**出现的 `ANSWER:` 行（兼容模型思考中多次写出）；
   - `expandToken` 把 `B`/`AC`/`12`/`1 3` 等字母/数字/连写形式统一展开为选项序号，再映射为**选项正文**（与题库存储约定一致：入库答案不带 A/B 前缀）；映射失败回退去前缀原文，绝不让答案为空。
10. `sendModelResponseToBackend(JSON.stringify({answer}))` 回传后端。

### 6.4 普通答题路径的兜底（`callModelAPI`, L2587+）

非 URL 题走 `'answer'` phase：`buildAnswerChatMessages`（`answerFewShot.ts (128-143)`）注入 system 规则 + 按题型的 few-shot 例题。其中：
- `shouldAttachAnswerFewShot`（L100-107）**显式排除 `__URL_QUESTION__:`**——图片题不注入文字 few-shot（走的是 6.3 的独立 prompt 体系）；
- `callModelAPI` 内还有一道 `hasImage` 检测（L2596：图片扩展名 URL 或 base64 关键字），命中时把视觉模型加入并发答题组。注意该兜底路径 query 是**纯文本**（URL 仅作为文本传给模型），属于防御性设计，实际视觉能力依赖 6.3 主路径。

---

## 7. 图片像素级预处理流水线（canvas，`Home.vue` + `questionImage.ts`）

| 函数 | 位置 | 作用 |
|---|---|---|
| `applyWhiteBackgroundToDataUrl` | Home.vue (3799-3834) | 透明 PNG 先画白底再叠图，输出 PNG data URL。**一箭双雕**：VLM 对透明背景更稳；UI 深色主题下不出现"黑底透明图" |
| `ensureDataUrlMinimumSize` | Home.vue (3836-3875) | 短边 < N 的图等比放大（白底填充），N 默认 32（`DEFAULT_VISION_IMAGE_MIN_SIZE`, L3796）——规避 VLM 最小尺寸限制 |
| `extractVisionImageSizeError` | Home.vue (3877-3891) | 从模型报错文本解析出所需最小尺寸（供 6.3-7 自动重试） |
| `shouldInvertTransparentDarkImage` | questionImage.ts (460-527) | **暗色模式适配的像素启发式**：canvas 采样 32×32，统计透明像素占比（alpha<245）与可见像素暗色占比（加权亮度≤110），当 `透明占比≥0.1 且 暗色占比≥0.6` 时判定为"透明底黑图"，返回 true |

最后一项配合 CSS 类 `invert-on-dark`（QuestionList.vue L571 / QuestionDetail.vue L220）：很多平台公式图是透明底黑字 PNG，浅色模式正常，深色主题若直接显示会"黑底黑字"不可读；检测命中后**仅在暗色主题下反色**。这是很细的产品级体验打磨。

---

## 8. 多模态协议适配（`src/services/modelProtocol.ts`）

VLM 消息统一以 OpenAI Chat 风格构建（`{type:'image_url', image_url:{url, detail}}`），由协议模板翻译为各 API 格式：

| 协议 | 图片格式转换 | 位置 |
|---|---|---|
| openai-chat | 原样透传 `image_url`（`detail:'high'` 由组装层指定） | L459-498 |
| openai-response | `mapResponsesContent`：`image_url` → `{type:'input_image', image_url, detail}` | L278-308 |
| anthropic | `mapAnthropicContent`：**data URL 正则拆解**为 `{type:'image', source:{type:'base64', media_type, data}}`；同时 system 消息从 messages 中抽出合并 | L364-416 |
| custom | 直接执行用户配置的 JS 代码（`jsCode`） | L728-730 |

即**内部多模态只有一种规范形**（OpenAI 风格），协议差异被隔离在这一层，上游 `buildMultimodalContent` 无感知——值得学习的适配器模式。

另：视觉模型连通性测试（`ModelSettings.vue (1392-1491)`）用内置 `vlm_test.png` 转 base64 构造 `image_url (detail:'low')` 测试消息，图片加载失败时回退一张 1×1 内联 base64 PNG（L1465-1466），保证测试永远可执行。

---

## 9. UI 展示层：题库中的含图题目如何渲染

### 9.1 列表与详情（`QuestionList.vue` / `QuestionDetail.vue`）

- 三个字段**全部**走 `splitQuestionImageParts` 拆段渲染（QuestionDetail.vue L195-197）：
  ```html
  <template v-for="(part, i) in contentParts">
    <span v-if="part.type === 'text'">{{ part.text }}</span>
    <img v-else-if="imgSrc(part.url)" :src="imgSrc(part.url)" :class="['question-image', invertClass(...)]"/>
    <span v-else class="image-loading">[图片加载中]</span>
  </template>
  ```
  `question / options / answer` 一视同仁——答案也可能含图（主观题答案贴图）。
- 图片 URL 收集：`visibleImageUrls`（List L573-584）/ `imageUrls`（Detail L212-217）对当前可见题目去重合并，再逐个 `fetchQuestionImageBase64` 物化。
- **两级缓存**：QuestionList 用**模块级** `_imageCache`（L563，组件销毁仍保留，翻页回来不重新抓）+ Rust 端磁盘缓存；QuestionDetail 直接复用磁盘缓存。
- `analyzeImage` → `shouldInvertTransparentDarkImage` → `invertClass` 暗色反色（L608-615）。
- 图片 `@load` 后触发布局重算（自定义滚动条 thumb 更新，Detail L44/L59）——异步图片高度变化对滚动的兼容。

### 9.2 请求详情面板（Home.vue）

- `urlQuestion.renderedHtml`：`buildRenderedHtml`（L3980-3999）把题面文本的 URL 替换成 **base64 内联 `<img>`**（带白底样式）+ 换行转 `<br/>`，`v-html` 渲染（L298-299）；失败 URL 显示 `[图片: url]` 文字占位。
- 下方同时流式展示 VLM 的分析过程与思考内容（L308-327），体验上让"看不见图的等待期"变得可解释。

### 9.3 编辑器

`QuestionEditor`/详情编辑表单（QuestionDetail.vue L98-147）就是普通 textarea，编辑的是**含 URL 的原始文本**；保存后 computed 自动重新拆段，新贴的图片 URL 立即可见。无图片上传/粘贴能力——URL 文本模型的一致性延伸。

---

## 10. 导入导出链路：图片如何"过边界"

### 10.1 导出（`src/utils/exporter.ts`）

CSV/XLSX/DOCX/TXT 全部**原样导出 URL 文本**（图片不嵌入文件）。PDF（`generatePDF`, L138-231）做了特殊设计：

- **隐写往返机制**：把整表 `questions` 序列化为 JSON → `btoa(unescape(encodeURIComponent(...)))` 处理中文 → 包裹 `<<<ZERROR_DATA_START>>>...<<<ZERROR_DATA_END>>>`，**新起一页、白色、1 号字**分块（每 1000 字符）写入（L201-225）。
- 目的：PDF 的"可视表格"仅供人阅读（图片 URL 可能被单元格换行截断、不可复制），而**导入方优先读隐写数据**即可 100% 无损还原（包括含 URL 的长题干）。

### 10.2 导入（`src/utils/importer.ts`）

- `parsePDF`（L36-127）：**先找隐藏数据标记**（L54-78），找到即整表解码导入；找不到才回退纯文本启发式（Y 坐标聚合行 + 从行尾倒序剥离 时间→类型→答案→ID 的正则，L222-310，非常脆弱，仅兜底）。
- DOCX（L347-411）：mammoth 转 HTML 找表格→逐单元格抽文本（保留 `<p>`/`<br>` 换行）；**内嵌图片会被 `getCellText` 剥标签丢弃**。
- CSV/XLSX/TXT：纯文本解析。
- 外部文件预览（`src/utils/fileReader.ts`）：Excel/DOCX/RTF/伪 doc(HTML)/二进制 doc 逐级嗅探+启发式文本提取，同样只出文本。

**边界结论**：导入导出链路上图片只能以"URL 文本"幸存；任何以二进制形式内嵌在 Word/PDF 里的图片都会被丢弃。这与第 3 节的存储模型完全自洽——图片的唯一权威来源是学习平台页面里的 URL。

---

## 11. 工程亮点总结（可复用的模式）

1. **延迟物化（Lazy Materialization）**：存储层只存"指向"，像素在使用点按需生成（URL→base64），并用两级缓存摊薄成本。数据模型因此保持极简。
2. **策略热更新 + 编译期验证 + 逐级回退**（questionImage.ts）：云控代码必须通过 preview 用例才启用；remote→local、算法→fallback、调用→try/catch 三层兜底，任何一层失效都不影响主流程。
3. **协议适配器**（modelProtocol.ts）：内部统一 OpenAI 多模态格式，差异封装在模板层，新增协议只改一处。
4. **反防盗链工程**（commands.rs）：按平台定制 Referer 的三级请求策略 + 文件头嗅探 MIME + 磁盘缓存。
5. **VLM 兼容性流水线**：白底合成 → 最小尺寸 → 发送前预检 → 失败解析错误自动放大重试；以及"下载失败即失败"的强一致策略（宁可不答，不容忍残题）。
6. **URL 在检索中的双面处理**（database.rs）：精确门禁要求 URL 集合相等（防配图不同的同文题误命中），相似度计算又把 URL 归一为占位符（防 URL 干扰正文比较）。
7. **像素级暗色适配启发式**（shouldInvertTransparentDarkImage）：用 32×32 采样 + 透明/暗色双阈值判定透明黑图，只在此类图上做暗色反色。
8. **PDF 隐写往返**：导出方写隐藏结构化数据、导入方优先读取，解决"人类可读格式不可靠"的往返完整性问题。
9. **双通道事件 + 幂等去重**：SSE 与 Tauri event 并发投递 `model-call-request`，`requestId:phase` Set 去重保证恰好一次执行。

---

## 12. 局限与潜在改进点（批判性视角）

1. **图片缓存以 URL 为键**（`hash(url).b64`，commands.rs L101-105）：URL 带 token/时效参数变化即缓存失效；且 `DefaultHasher` 非内容哈希、无过期清理，缓存目录只增不减。改进方向：内容寻址（对图片字节做哈希）+ LRU/TTL。
2. **`buildRenderedHtml` 未做 HTML 转义**（Home.vue L3980-3999）：题干文本来自 OCS 请求（外部输入），直接拼接进 `v-html`，理论上存在注入面（Tauri WebView 内危害有限，但值得一层 escape）。
3. **兜底路径视觉模型看不了图**：`callModelAPI` 的 `hasImage` 分支（L2596-2600）把视觉模型加入并发，但消息仍是纯文本——模型只能"读 URL 字符串"，此分支实际意义有限。
4. **导入丢失二进制图片**：DOCX/PDF 中内嵌图片在导入时被丢弃，导入含图题的唯一途径是文本里带可访问的 URL。
5. **遗留兼容代码**：`checkAndShowUrlDialog`（Home.vue L3543+，检测响应 `题目中含有URL，无法直接展示`）对应的旧后端行为已不存在于当前 Rust 代码中，属死代码路径，可清理。
6. **图片在编辑体验上的空白**：不支持粘贴/上传图片（自动转成本地缓存并插入占位 URL），用户只能手工贴 URL 文本。

---

## 13. 关键源码索引（速查表）

| 关注点 | 文件 | 关键位置 |
|---|---|---|
| 图片 URL 识别/拆分/云控算法 | `src/utils/questionImage.ts` | 全文件（527 行） |
| 远程目录（算法下发） | `src/services/modelConfig.ts` | L96-101, L160-166, L339-360 |
| 图片抓取/缓存/MIME 嗅探 | `src-tauri/src/commands.rs` | L92-294 |
| URL 检测/路由/等待预算 | `src-tauri/src/server.rs` | L45-93, L346-362, L378-653 |
| 题库匹配中的 URL 处理 | `src-tauri/src/database.rs` | L13-39, L221-250, L1370-1408 |
| 视觉分析主流程 | `src/views/Home.vue` | L3440-3540, L4086-4281 |
| 多模态组装/图片预处理 | `src/views/Home.vue` | L3796-4082 |
| 模型调用路由/few-shot 排除 | `src/views/Home.vue` + `src/utils/answerFewShot.ts` | L2553-2585 / L100-143 |
| 题型分类/选项解析/答案解析 | `src/utils/urlQuestion.ts` | 全文件 |
| 多模态协议适配 | `src/services/modelProtocol.ts` | L278-416, L697-743 |
| 列表/详情图片渲染 | `src/views/questions/QuestionList.vue`, `QuestionDetail.vue` | List L563-640; Detail L192-217, L421-446 |
| 视觉模型测试图 | `src/views/settings/ModelSettings.vue` | L1392-1491 |
| PDF 隐写导出/导入 | `src/utils/exporter.ts`, `importer.ts` | exporter L138-231; importer L36-127 |
