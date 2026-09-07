---
name: agy-coding
description: "Drive Google Antigravity CLI (agy) to edit a repo. Pick the model, write the task, check the envelope, run verification, read the diff, loop, and commit."
---

# agy 上手指南（给数字员工）

目标：看完就能稳定驱使 Google Antigravity CLI（`agy`）改代码，少踩坑。  
官方文档：https://www.antigravity.google/docs/cli/

`agy` 是会改仓库的编码代理。你是调用方：选模型、写任务、核结果、跑验证、读 diff、提交。不要把它当黑盒一键完工。

---

## 0. 它是什么、你是谁

| | 做什么 | 不做什么 |
| --- | --- | --- |
| **agy** | 按 prompt 改工作树里的源码与测试 | 替你选产品对错、替你验收、替你负责上线 |
| **你（调用方）** | 定目标与禁令、选模型、核信封、本地验证、读 diff、commit | 默认信任它的「已自测」；在它改完后大片手改产品代码再 resume |

原则：

1. 它写代码，你对工作树负责。
2. SUCCESS 只表示它认为回合走完了，不等于类型过、测试过、点击活着。
3. 一次一个薄切片；返工用同一会话续跑，不要并行两个 agy 打同一棵树。

---

## 1. 安装与第一次能跑

1. 安装 CLI（以官方为准），确认 `agy` 在 PATH 里。
2. **先 `cd` 进目标仓库根目录**，再启动。它认的是当前工作树。
3. 本机第一次：完成 Google / 密钥认证；确认可用模型列表：

```bash
agy models
```

4. 若用 API key：仅设 key 不够，还要在 CLI settings 里把 `modelProvider` 设成对应提供方（例如 `gemini`）。未认证时 headless 会直接失败。
5. 仓库根可放 `AGENTS.md` / `GEMINI.md`：项目纪律（禁 commit、禁验收话术、架构约束）。启动时会读，比只写在某一轮 prompt 里稳。

---

## 2. 最小可跑命令（先会这个）

把任务写进文件，再喂给 `-p`（避免 shell 转义吃掉约束）：

```bash
cd /path/to/repo

agy --model <latest-flash-or-pro-highest-thinking> \
  --effort high \
  --dangerously-skip-permissions \
  --print-timeout 20m \
  --output-format json \
  -p "$(cat /tmp/agy-task.md)"
```

模型怎么选见第 4 节。下面示例里的 slug 只是占位，以 `agy models` 当时最新为准。

### 硬约束（背下来）

1. **`--model` 必须在 `-p` 前面。** 写反了，模型可能被吞掉，你以为在用 Pro，实际是默认模型。
2. **未知 model slug 会大声失败**——先 `agy models`，不要凭记忆拼名字。
3. **`--print-timeout` 默认约 5 分钟。** 正经改代码给 **15–20 分钟**。超时后先看 `git status` / `git diff`：**超时 ≠ 没写文件**，树里可能已经落地，再决定 resume 还是提交。
4. **`--output-format json`**，只信信封字段，不信长篇自我表扬。
5. **可信、隔离的本机工作树**才用 `--dangerously-skip-permissions`。原因见下一节。

### 读信封

只认这些：

- `status == "SUCCESS"`（字符串全等）
- `conversation_id`（立刻存下来，续跑用）
- `error`（有内容就当失败）

其余叙述当噪音。

---

## 3. 权限：为什么「exit 0」却什么都没改

- 工作区文件读写通常自动放行。
- **Shell 默认是 Ask。** Headless / 无人值守时 Ask 会变成 soft-deny：进程仍可能 exit 0，信封甚至 SUCCESS，但 `git` / 包管理器 / 测试命令根本没跑，工作树零变化。

处理：

- 可信环境：加 `--dangerously-skip-permissions`；或
- 在 CLI settings 里只 allow 你需要的命令（如 git、npm、npx、node）。

「SUCCESS + 文件没动」时，先查权限与认证，再怪模型。

---

## 4. 选模型：升级链（思考强度一律最高）

先跑 `agy models`，在列表里认最新档位，**不要死记旧 slug**（版本号会变）。凡带思考/effort 选项的，**一律拉到最高**（例如 slug 带 `-high`，或另加 `--effort high`——以你当前 CLI 实际支持为准）。

### 默认升级链（背这个）

```
① 最新 Flash（思考最高）
      ↓  同一问题 miss 两次，或明显假修 / 边界搞不定
② 最新 Pro（思考最高）
      ↓  同一问题 Pro 仍 miss
③ Claude（思考最高；Sonnet / Opus 以当时列表里最强可用为准）
```

- **开头默认用最新 Flash、思考最高。** 新 UI、常规实现都从这里开。
- **不要在同一档上无限重试。** 另一次同 slug 重跑不是升级。
- **换档 = 新会话**（换 `--model`），并在新 prompt 写清上一档失败证据；不要假装还在旧 conversation 里换脑。
- **配额紧的档（常见是 Claude）省着用**：只在 Flash→Pro 都搞不定同一问题时升。

### 什么时候直接从 Pro 开（可跳过 Flash）

已知是硬 bug、点击/路由死、schema、打包/客户端服务端边界、你已经有「Flash 会糊胶水」的预期时，可以直接最新 Pro（思考最高）。其余仍从 Flash 开。

### 假修信号（看到就升档，别再 Flash）

用 history API 假装导航、在 capture 阶段拦事件自建分发、测试里 mock 掉事件导致「单测绿、真鼠标死」、只改表象不改真实事件路径。同一类信号出现两次，立刻升到链上下一档。

### 命令里怎么写

```bash
# 1) 看当时真正可用的最新 slug
agy models

# 2) 选用「最新 Flash + 思考最高」这类 slug（名称以 models 输出为准）
agy --model <latest-flash-highest-thinking>   --effort high   --dangerously-skip-permissions   --print-timeout 20m   --output-format json   -p "$(cat /tmp/agy-task.md)"
```

若当前 CLI 不支持 `--effort`，就选名字里已带最高思考的 slug，不要再选 medium/low。

---

## 5. 怎么写 prompt（由浅入深）

### 浅：一个可观察结果

写清：

1. **目标**：用户能看见/能点出的结果（不是「改某某函数」清单）。
2. **范围**：建议改/必核的路径；写明「以现有实现为准，不要平行复制一套」。
3. **验证**：你本地会跑的命令（tsc、测试、必要的点击路径）。
4. **禁令**：不 commit、不推远程、不写验收话术、不动无关文件、不引入项目禁止的模式。

把以上放进 `/tmp/agy-task.md`，用第 2 节命令启动。

### 中：切片与续跑

- 一次一个薄切片。做完核验，再开下一块（新 prompt）；同一块返工用 `--conversation`。
- 续跑 prompt 只写增量：哪红了、点了哪里死、禁止再走哪条假修。不要整篇重贴还扩范围。
- **同一产品线可以跨多天 resume 同一个 `conversation_id`（同一模型）**，上下文里的约束会跟着走，通常比频繁新开便宜。
- 要换模型：开新会话，并在新 prompt 写清上一轮失败证据。
- **同一工作树同一时刻只跑一个 agy。** 要并行就第二份 clone。

续跑骨架：

```bash
agy --model <same-model-as-first-turn> \
  --conversation "$CONVERSATION_ID" \
  --effort high \
  --dangerously-skip-permissions \
  --print-timeout 20m \
  --output-format json \
  -p "$(cat /tmp/agy-task-resume.md)"
```

续跑必须同一模型。要升级 Flash→Pro→Claude 时，开新会话，不要在旧 conversation 上换模型。

### 深：大改与边界类任务

架构级变更（路由形态、鉴权平面、打包边界）：

1. **先由调用方锁一份可执行方案**（目标、非目标、验收口径、禁止方案），再按切片喂 agy。让它在 prompt 里现场发明生产架构，容易写出「开发像对、生产不像」的方案。
2. **客户端可达代码不要同步拉进仅服务端依赖**（数据库驱动、ORM、Node-only API）。服务端逻辑放明确的 server-only 模块，或只在服务端 handler 内动态 import。
3. **URL / 宿主相关逻辑**：若产品要求某类主机上的地址形态固定，在禁令里写死；所有导航与链接走统一封装，页面要能读到正确的宿主上下文，否则默认值会悄悄写错路径。
4. **修完一类泄漏，扫同目录兄弟文件**，并加便宜的源码级断言，防止下一轮又写回硬编码路径。
5. **环境差异写进禁令**：若某框架在生产与本地对「改写请求 URL」行为不同，禁止把仅本地有效的 rewrite 当最终方案。

---

## 6. 跑完之后：你的核验闭环（不可省）

```
写 prompt 文件
  → cd 仓库 + 正确 flag 启动
  → 核信封（SUCCESS / 存 conversation_id；超时则先看 diff）
  → 你自己跑类型检查与测试
  → 真浏览器点相关路径（涉及共享前端包时看 Console，确认没有服务端运行时进客户端包）
  → 不过：同会话 resume，或按规则换模型
  → 过：你读 diff → 丢掉垃圾 → 你自己 commit
  → 需要上线：本地核验绿之后再部署（用户喊停就停）
```

### diff 里丢掉什么

- `.agy-*`、本地会话缓存
- 临时 `fix_*.sh`、inject 探针、一键修复脚本
- 顺手生成的验收文档、无关 README、截图目录（除非任务明确要求）

### 提交纪律

- **你 commit，agy 不 commit。** prompt 里写禁，diff 里若出现它写的提交脚本，删掉。
- **不要大片手改产品代码再 resume**（会话与树会分叉）。要改就写进下一轮 prompt。  
  例外：删除会把服务端依赖重新引进客户端的**一行未使用 import**，可以你删，并在下一轮禁令里写明。
- 读 diff 时盯：假导航/假点击、越权改配置、硬编码单一客户、切片外文件、客户端碰到服务端依赖、错误主机上的错误 URL。

---

## 7. 避坑清单（通用）

| 现象 | 常见原因 | 你怎么做 |
| --- | --- | --- |
| SUCCESS，树零变化 | shell soft-deny / 未认证 | 权限跳过或 allowlist；查登录与 provider |
| 以为在用 Pro | `--model` 写在 `-p` 后 | 模型 flag 永远在前 |
| 改动残缺或「跑完没改完」 | 默认 5m 超时 | 15–20m；超时后先看 diff |
| 超时当失败扔掉 | 误以为超时=回滚 | 查工作树；有落地则验测或 resume |
| 单测绿、真点击死 | 假修导航/事件；或客户端包进了服务端运行时 | 真点；升 Pro；查打包边界 |
| diff 错乱 | 两台 agy 同树并行 | 同时只跑一个 |
| resume 对不上 | 中途手改产品代码 | 用 prompt 改，少手改 |
| 提交夹带垃圾 | 让 agy commit | 你读 diff 后自己提交 |
| 生产与本地行为不一致 | 把仅本地有效的 rewrite/假设当方案 | 合同写死生产约束；生产构建或等价环境验证 |
| 修了一个跳转，下一跳又炸 | 同类硬编码 / 缺上下文 | 按类扫描；加源码断言 |
| 半成品已上线 | 急着部署 | 本地真点绿再上；用户停就停 |

---

## 8. 开跑前 / 跑完后口令

**开跑前**

- [ ] 已 cd 到正确仓库
- [ ] `agy models` 确认 slug
- [ ] `--model` 在 `-p` 前
- [ ] timeout ≥ 15m
- [ ] 可信环境权限已处理
- [ ] 认证与 modelProvider 可用
- [ ] prompt 在文件里：结果 + 验证 + 禁令
- [ ] 本树没有另一个 agy
- [ ] 按第 4 节升级链选模型；思考强度最高；同档不无限重试
- [ ] 大架构已有锁死方案（若适用）

**跑完后**

- [ ] SUCCESS 且 error 空；若超时，已看过 diff
- [ ] `conversation_id` 已存
- [ ] 类型检查与测试你自己跑过
- [ ] 真点击过；必要时查客户端包是否干净
- [ ] diff 无 `.agy` 垃圾、无假修、无越权、无错误边界
- [ ] 你自己 commit
- [ ] 需要部署时，本地已绿

哪一格打不了勾就停。不要用「再跑一次同档 Flash」填空。

---

## 9. 常见误读

- 「SUCCESS 就是做完了。」不是。还要你的测试与真点击。
- 「超时了所以什么都没写。」不一定。先看 diff。
- 「再跑一次同样的 Flash 会好。」通常不会；按 Flash→Pro→Claude 升级，思考始终最高。
- 「设了 API key 就能跑。」还要 provider、登录/keyring、shell 真被允许。
- 「prompt 越细到每一行 patch 越好。」要可观察结果与禁令；大架构用单独方案文档。
- 「两个切片并行两个 agy 更快。」同树不行。
- 「我先手改两行再 resume。」尽量不要。
- 「让它顺便 commit。」不要。
- 「接口/单测绿就能上。」涉及 UI 与打包边界时不够。

---

## 10. 建议阅读顺序（由浅入深）

1. 第 0–2 节 → 能发出第一次 headless 调用  
2. 第 3–4 节 → 权限与选模型不再猜  
3. 第 5 节浅/中 → 稳定切片与续跑  
4. 第 6–8 节 → 核验与提交成为肌肉记忆  
5. 第 5 节深 + 第 7 节 → 架构、边界、生产差异类任务  

官方变更以官方为准。本文是调用方实务备忘：坑来自真实驱使，不随文档措辞消失。
