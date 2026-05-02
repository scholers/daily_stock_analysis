# LLM (大模型) 配置指南

欢迎！无论你是刚接触 AI 的新手小白，还是精通各种 API 的高玩老手，这份指南都能帮你快速把大模型（LLM）跑起来。

本项目对外提供统一的 AI 模型接入体验，支持主流官方 API、OpenAI 兼容平台以及本地模型。底层由 [LiteLLM](https://docs.litellm.ai/) 驱动，但大多数用户只需要理解“选服务商、填 API Key、选主模型/渠道”这条默认路径。为了照顾不同阶段的用户，我们设计了“三层优先级”配置，按需选择最适合你的方式即可。

---

## 快速导航：你应该看哪一节？

1. **【新手小白】** "我只想赶紧把系统跑起来，越简单越好！" -> [指路【方式一：极简单模型配置】](#方式一极简单模型配置适合新手)
2. **【进阶用户】** "我有好几个 Key，想配置备用模型，还要改自定义网址(Base URL)。" -> [指路【方式二：渠道(Channels)模式配置】](#方式二渠道channels模式配置适合进阶多模型)
3. **【高玩老手】** "我要做复杂的负载均衡、请求路由、甚至多异构平台高可用！" -> [指路【方式三：YAML 高级配置】](#方式三yaml高级配置适合老手自定义)
4. **【本地模型】** "我想用 Ollama 本地模型！" -> [指路【示例 4：使用 Ollama 本地模型】](#示例-4使用-ollama-本地模型)
5. **【视觉模型】** "我想用图片识别股票代码！" -> [指路【扩展功能：看图模型(Vision)配置】](#扩展功能看图模型vision配置)

---

## 方式一：极简单模型配置（适合新手）

**目标：** 只要记得填入 API Key 和对应的模型名就能立刻用。不需要折腾复杂概念。

如果你只打算用一种模型，这是最快捷的办法。打开项目根目录下的 `.env` 文件（如果没有，复制一份 `.env.example` 并重命名为 `.env`）。

### 示例 1：使用通用第三方平台（兼容 OpenAI 格式，推荐）

现在市面上绝大多数第三方聚合平台（例如硅基流动、AIHubmix、阿里百炼、智谱等）都兼容 OpenAI 的接口格式。只要平台提供了 API Key 和 Base URL，你都可以按照以下格式无脑配置：

```env
# 填入平台提供给你的 API Key
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx
# 填入平台的接口地址 (非常重要：结尾通常必须带有 /v1)
OPENAI_BASE_URL=https://api.siliconflow.cn/v1
# 填入该平台上具体的模型名称（非常重要：注意前面必须加上 openai/ 前缀帮系统识别）
LITELLM_MODEL=openai/deepseek-ai/DeepSeek-V3 
```

### 示例 2：使用 DeepSeek 官方接口
```env
# 填入你在 DeepSeek 官方平台申请的 API Key
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxx
```
*兼容提示：仅填这一行时，系统仍会默认使用 `deepseek/deepseek-chat` 并在日志提示迁移。*
`deepseek-chat` / `deepseek-reasoner` 仍可用于兼容旧配置，但 DeepSeek 官方已标记为 2026/07/24 后废弃；新配置建议通过 Web 快速渠道或显式 `LITELLM_MODEL=deepseek/deepseek-v4-flash` 迁移到 `deepseek-v4-flash` / `deepseek-v4-pro`。

### 示例 3：使用 Gemini 免费 API
```env
# 填入你获取的 Google Gemini Key
GEMINI_API_KEY=AIzac...
```

### 示例 4：使用 Ollama 本地模型
```env
# Ollama 无需 API Key，本地运行 ollama serve 后即可使用
OLLAMA_API_BASE=http://localhost:11434
LITELLM_MODEL=ollama/qwen3:8b
```

> **重要**：Ollama 必须使用 `OLLAMA_API_BASE` 配置，**不要**使用 `OPENAI_BASE_URL`，否则系统会错误拼接 URL（如 404、`api/generate/api/show`）。远程 Ollama 时，将 `OLLAMA_API_BASE` 设为实际地址（如 `http://192.168.1.100:11434`）。当前依赖要求 LiteLLM ≥1.80.10（与 requirements.txt 一致）。

> **恭喜！小白读到这里就可以去运行程序了！**
> 想测测看通没通？在主目录打开命令行输入：`python test_env.py --llm`

---

## 方式二：渠道(Channels)模式配置（适合进阶/多模型）

**目标：** 我有多个不同平台的 Key 想要混着用，如果主模型卡了/网络挂了，我希望它能自动切换到备用模型。

**网页端可以直接配：** 你可以启动程序后，在 **Web UI 的“系统设置 -> AI 模型 -> AI 模型接入”** 中非常直观地进行可视化配置！

> **新版编辑体验补充**：对于 DeepSeek、阿里百炼（DashScope）以及其他兼容 OpenAI `/v1/models` 的渠道，设置页现在支持直接点击“获取模型”，从 `{base_url}/models` 拉取可用模型并多选；底层仍会保存为原来的 `LLM_{CHANNEL}_MODELS=model1,model2` 逗号格式。若渠道不支持该接口、鉴权失败或暂时不可达，仍可继续手动填写模型列表，不影响保存。

### 首次启动配置状态

后端提供只读状态接口 `GET /api/v1/system/config/setup/status`，用于判断首次启动闭环中最基础的几类配置是否已经就绪：LLM 主渠道、Agent 渠道、自选股、通知渠道和本地存储。这个接口只读取已保存的 `.env` 与当前进程环境变量，不会重载运行时配置、写入 `.env`、测试真实模型或创建数据库文件；前端向导和后续 smoke run 可以基于该接口逐步接入。

### Web 渠道编辑器的兼容性 / 迁移 / 回退规则

- 预设里的 provider / Base URL / 示例模型只用于**初始化表单**；真正落盘时仍是你当前输入的 `LLM_{CHANNEL}_PROTOCOL`、`LLM_{CHANNEL}_BASE_URL`、`LLM_{CHANNEL}_MODELS`、`LLM_{CHANNEL}_API_KEY(S)`，不会在后台偷偷改成别的 provider 名或 URL。
- 设置页的“获取模型”只对 `OpenAI Compatible` / `DeepSeek` 渠道调用 `{base_url}/models`；“测试连接”只发一次最小聊天请求。两者返回的 `stage / error_code / details / latency_ms` 仅用于结构化诊断提示，**不会写回** `.env`。
- 保存渠道时，只会更新这次提交的 key；不会因为切换渠道模式而静默迁移整个旧配置。唯一会被**同步清理**的是运行时模型引用：如果 `LITELLM_MODEL`、`AGENT_LITELLM_MODEL`、`VISION_MODEL` 或 `LITELLM_FALLBACK_MODELS` 指向了当前已启用渠道里已经不存在的模型，设置页会在保存前把这些失效引用清空/移除，避免运行时继续指向无效模型；像 `cohere/...` 这类直连环境变量 provider 仍会保留。
- 回退方式也保持最小：把对应渠道模型列表改回去后重新选择主模型 / fallback，或直接用桌面端导出备份 / 手动 `.env` 还原之前的 `LLM_*`、`LITELLM_MODEL`、`AGENT_LITELLM_MODEL`、`VISION_MODEL`、`LLM_TEMPERATURE` 即可，不需要额外跑迁移脚本。
- 当前仓库对此链路的依赖窗口是 `litellm>=1.80.10,<1.82.7`（见 `requirements.txt`）；回归覆盖包括 `tests/test_system_config_service.py`、`tests/test_system_config_api.py` 和 `apps/dsa-web/src/components/settings/__tests__/LLMChannelEditor.test.tsx`。

### 回退与兼容性证据

- 兼容窗口与静默清理范围：在 `litellm>=1.80.10,<1.82.7` 时，保存仅清理失效的 runtime 模型引用（`LITELLM_MODEL`、`AGENT_LITELLM_MODEL`、`VISION_MODEL`、`LITELLM_FALLBACK_MODELS`），`cohere/*` 等非渠道直连模型会被保留。
- 回退方式：可直接用桌面端导出备份后通过 `POST /api/v1/system/config/import` 恢复；也可手动把 `.env` 中历史 `LITELLM_* / AGENT_LITELLM_MODEL / VISION_MODEL / LLM_TEMPERATURE` 回填后重启生效。
- 回退回归证据：`tests/test_system_config_service.py::test_import_desktop_env_restores_runtime_models_after_cleanup` 覆盖“清理后用桌面导出备份恢复 runtime 引用”。
- 建议回退操作链路（含设置页刷新）：先导出桌面备份，`POST /api/v1/system/config/import` 导入后，再通过 `GET /api/v1/system/config` 刷新页面配置，再确认 `LITELLM_MODEL / AGENT_LITELLM_MODEL / VISION_MODEL / LLM_TEMPERATURE` 与模型列表一致后再继续使用。

### 常用官方文档来源（用于核对预设 provider / Base URL / 模型命名）

- OpenAI Compatible 规范（LiteLLM）：<https://docs.litellm.ai/docs/providers/openai_compatible>
- OpenAI 官方：<https://platform.openai.com/docs/api-reference/chat>
- DeepSeek 官方：<https://api-docs.deepseek.com/>
- 阿里百炼 DashScope 兼容模式：<https://help.aliyun.com/zh/model-studio/compatibility-of-openai-with-dashscope>
- Moonshot / Kimi 官方：<https://platform.moonshot.ai/docs/guide/compatibility>
- Anthropic 官方：<https://docs.anthropic.com/en/api/messages>
- Gemini 官方：<https://ai.google.dev/gemini-api/docs/openai>
- Ollama 官方：<https://github.com/ollama/ollama/blob/main/docs/api.md>

如果不方便用网页版，在 `.env` 文件中配置也非常丝滑，它能让你同时管理多个第三方平台。规则如下：

1. **先声明你有几个渠道**：`LLM_CHANNELS=渠道名称1,渠道名称2`
2. **给每个渠道分别填写配置**（注意全大写）：`LLM_{渠道名}_XXX`

### 示例：同时配置 DeepSeek 和某中转平台，并设置备用切换
```env
# 1. 开启渠道模式，声明这里有两个渠道：deepseek 和 aihubmix
LLM_CHANNELS=deepseek,aihubmix

# 2. 渠道一：配置 DeepSeek 官方
LLM_DEEPSEEK_BASE_URL=https://api.deepseek.com
LLM_DEEPSEEK_API_KEY=sk-1111111111111
LLM_DEEPSEEK_MODELS=deepseek-v4-flash,deepseek-v4-pro

# 3. 渠道二：配置一个常用的聚合中转 API
LLM_AIHUBMIX_BASE_URL=https://api.aihubmix.com/v1
LLM_AIHUBMIX_API_KEY=sk-2222222222222
LLM_AIHUBMIX_MODELS=gpt-4o-mini,claude-3-5-sonnet

# 4. 【关键】指定主模型和备用模型列表
# 平时首选用 deepseek 这款模型：
LITELLM_MODEL=deepseek/deepseek-v4-flash
# 可选：Agent 问股单独指定主模型（留空则继承主模型）
AGENT_LITELLM_MODEL=deepseek/deepseek-v4-pro
# 主模型崩了立刻挨个尝试下面这俩备用模型：
LITELLM_FALLBACK_MODELS=openai/gpt-4o-mini,anthropic/claude-3-5-sonnet
```

### 示例：Ollama 渠道模式（本地模型，无需 API Key）
```env
# 1. 开启渠道模式，声明 ollama 渠道
LLM_CHANNELS=ollama

# 2. 配置 Ollama 地址（本地默认 11434 端口）
LLM_OLLAMA_BASE_URL=http://localhost:11434
LLM_OLLAMA_MODELS=qwen3:8b,llama3.2

# 3. 指定主模型
LITELLM_MODEL=ollama/qwen3:8b
```

### MiniMax 渠道模型填写说明

- 如果你通过 OpenAI Compatible 渠道接 MiniMax，请在渠道模型里直接填写 `minimax/<模型名>`，例如 `minimax/MiniMax-M1`。
- Web 设置页里的主模型、Agent 主模型、Fallback、Vision 下拉会保留这个值原样展示，不会再错误改写成 `openai/minimax/<模型名>`。

### 问股 Agent / LiteLLM 配置兼容说明

- 问股 Agent 运行时沿用与普通分析相同的三层优先级：`LITELLM_CONFIG`（LiteLLM YAML）> `LLM_CHANNELS` > legacy provider keys。只要上层配置有效生效，下层配置就不会再参与本次请求。
- YAML 模式下，Agent 直接复用 LiteLLM `model_list` / `model_name` 路由语义；渠道模式下，优先读取 `AGENT_LITELLM_MODEL`，留空时继承 `LITELLM_MODEL`，再按 `LITELLM_FALLBACK_MODELS` 继续 fallback。
- 如果你没有启用 YAML / Channels，且 `AGENT_LITELLM_MODEL` 也留空，但本地仍保留 legacy 环境变量，问股 Agent 依然会继承旧配置：`GEMINI_API_KEY + GEMINI_MODEL` -> `gemini/<model>`，`OPENAI_API_KEY + OPENAI_MODEL` -> `openai/<model>`，`ANTHROPIC_API_KEY + ANTHROPIC_MODEL` -> `anthropic/<model>`。
- 本次修复只增强“失败时保留后端真实错误原因”和“未配置 LLM 时给出更具体诊断”，**不会**静默删除、清空、迁移或改写你现有的 `GEMINI_*` / `OPENAI_*` / `ANTHROPIC_*` / `LITELLM_*` 配置。
- 如果当前环境没有任何有效 Agent 模型链路，问股页面会继续按失败语义返回，并直接展示后端真实配置诊断；补齐任一有效模型来源后即可恢复，无需额外执行配置迁移脚本。
- 推荐的新配置方式仍然是显式设置 `LITELLM_MODEL` / `AGENT_LITELLM_MODEL` 或使用 `LLM_CHANNELS`；legacy provider keys 目前保留为兼容回退路径，方便旧 `.env`、本地 macOS 开发环境和历史部署平滑继续运行。

### Kimi K2.6 固定 temperature 兼容说明

- Moonshot 官方说明 Kimi API 兼容 OpenAI 接口，Base URL 使用 `https://api.moonshot.ai/v1`：<https://platform.kimi.ai/docs/guide/kimi-k2-6-quickstart>
- LiteLLM 官方要求 OpenAI Compatible 渠道模型名使用 `openai/` 前缀：<https://docs.litellm.ai/docs/providers/openai_compatible>
- Moonshot 官方兼容性文档区分两种固定值：**thinking 模式固定 `1.0`，non-thinking 模式固定 `0.6`**；传其它值会被接口拒绝：<https://platform.moonshot.ai/docs/guide/compatibility#parameters-differences-in-request-body>
- 当前仓库的运行时依赖窗口是 `litellm>=1.80.10,<1.82.7`（见 `requirements.txt`）；本次兼容逻辑按该范围回归验证了主分析、大盘复盘、Agent 直连 LiteLLM，以及系统设置页的渠道连通性测试。
- 因此本项目会在请求发出前按**实际请求模式**归一化 `kimi-k2.6` 及其 `kimi-k2.6-*` 变体：默认 / thinking 路径使用 `temperature=1.0`；如果你的 LiteLLM YAML 路由别名里显式写了 `litellm_params.extra_body.thinking.type: disabled`（或等价 non-thinking 配置），则自动切到 `temperature=0.6`。你在 `.env` 或 Web 设置里保存的 `LLM_TEMPERATURE` 不会被改写。
- `SystemConfigService` 在 Web 设置保存 / 桌面端 `.env` 导入时只更新你提交的 key，不会因为切到 Kimi 静默清空、迁移或重写已有 `LLM_TEMPERATURE`；渠道测试请求里临时使用的 `1.0/0.6` 也不会回写到配置文件。
- 非 Kimi 主模型、非 Kimi fallback 以及切回普通模型后的请求，仍继续使用你配置的温度；也就是说旧配置无需迁移，切换模型即可自动恢复原行为。
- 本仓库兼容性回归覆盖见：`tests/test_llm_channel_config.py`、`tests/test_market_analyzer_generate_text.py`、`tests/test_agent_pipeline.py`、`tests/test_system_config_service.py`。
- 最小回滚方式：直接回退本次 Kimi 固定温度相关改动，无需单独迁移已有 `LLM_TEMPERATURE` 配置。

### 兼容性与回退复核清单（按 PR 审核口径）

- 运行时依赖窗口：`litellm>=1.80.10,<1.82.7`（与 `requirements.txt` 一致）。
- 回归验证入口：
  - 渠道模型发现与连接：`tests/test_llm_channel_config.py`
  - 运行时源清理与恢复（含桌面导出备份链路）：`tests/test_system_config_service.py`
  - 接口校验与问题面向字段：`tests/test_system_config_api.py`
  - 设置页交互与保存后提示：`apps/dsa-web/src/components/settings/__tests__/LLMChannelEditor.test.tsx`
- 旧配置回退路径：`桌面端导出备份 -> /api/v1/system/config/import`，或手动恢复 `LLM_* / LITELLM_* / AGENT_LITELLM_MODEL / VISION_MODEL / LLM_TEMPERATURE`。

> **致命避坑说明**：如果你启用了 `LLM_CHANNELS`，那么你直接写在外面的 `DEEPSEEK_API_KEY` 或 `OPENAI_API_KEY` 将**全部失效（系统一律无视）**！二者**选其一即可**，千万不要既写了新手模式又写了渠道模式结果产生冲突。
> **Docker 注意**：如果你在 `docker compose environment:` 或 `docker run -e` 中显式传入 `LITELLM_MODEL`、`LLM_CHANNELS`、`LLM_DEEPSEEK_MODELS` 等变量，容器重启后这些环境变量会覆盖 Web 设置页写入的 `.env`，需要同步修改部署配置。

---

## 方式三：YAML 高级配置（适合老手自定义）

**目标：** 我不在乎学习门槛，我要最高控制权，我要用原生规则做企业级高可用！

这一层会直接映射到底层 LiteLLM 路由能力，支持高并发、自动重试、按 RPM/TPM 负载均衡等操作。

### 本地运行 / Docker 部署模式配置说明

1. 在 `.env` 中只保留一行指向声明：
   ```env
   LITELLM_CONFIG=./litellm_config.yaml
   ```
2. 在项目根目录创建一个 `litellm_config.yaml`（可以参考自带的 `litellm_config.example.yaml`）。

示例 `litellm_config.yaml`：
```yaml
model_list:
  - model_name: my-smart-model
    litellm_params:
      model: deepseek/deepseek-v4-flash
      api_base: https://api.deepseek.com
      api_key: "os.environ/MY_CUSTOM_SECRET_KEY"  # 从环境变量读取 Key，安全防泄漏

  # Ollama 本地模型（无需 api_key）
  - model_name: ollama/qwen3:8b
    litellm_params:
      model: ollama/qwen3:8b
      api_base: http://localhost:11434
```

### GitHub Actions配置说明

1. `Settings` → `Secrets and variables` → `Actions`。非敏感配置（如模型名、开关、Base URL）可以放在 `Secret` 或 `Variables`；凡是 `*_API_KEY` / `*_API_KEYS` 以及 `LLM_<NAME>_API_KEY` / `LLM_<NAME>_API_KEYS` 这类密钥字段，请统一放在 `Secret` 标签页的 `New repository secret`

2. 按下表配置，只有全部必填配置正确配置，YAML 高级配置模式才可以生效，YAML配置文件的写法，可以参考自带的 `litellm_config.example.yaml`

| Secret 名称 | 说明 | 必填 |
|------------|------|:----:|
| `LITELLM_CONFIG` | 高级模型路由配置文件路径，通常配置 `./litellm_config.yaml` | 必填 |
| `LITELLM_MODEL` | 默认主模型名称或路由别名 | 必填 |
| `LITELLM_CONFIG_YAML` | 存放 YAML 配置文件内容，可不在仓库中提交实体文件 | 可选 |
| `LITELLM_API_KEY` | 用于存储API Key，可在配置文件中引用（环境变量引用方式）。由于GitHub Actions必须要指定导入的环境变量，因此你不能像本地运行模式那样自由命名环境变量 | 可选，必须配置到repository secret中 |
| `ANTHROPIC_API_KEY` | 如果要多个API Key，这个变量名称也能拿来用 | 可选，必须配置到repository secret中 |
| `OPENAI_API_KEY` | 同上，可以用来存储API Key | 可选，必须配置到repository secret中 |

渠道模式无需上传 YAML 文件。仓库自带 `daily_analysis.yml` 已显式透传以下常用字段：

- 运行时选择：`LLM_CHANNELS`、`LITELLM_MODEL`、`LITELLM_FALLBACK_MODELS`、`AGENT_LITELLM_MODEL`、`VISION_MODEL`、`VISION_PROVIDER_PRIORITY`、`LLM_TEMPERATURE`
- 多 Key：`GEMINI_API_KEYS`、`ANTHROPIC_API_KEYS`、`OPENAI_API_KEYS`、`DEEPSEEK_API_KEYS`（当前 workflow 仅从 repository secrets 导入，不会读取同名 Variables）
- 常用渠道名：`primary`、`secondary`、`gemini`、`deepseek`、`aihubmix`、`openai`、`anthropic`、`moonshot`、`ollama`

例如在 GitHub Actions 中配置 `LLM_CHANNELS=primary,deepseek` 时，需同步配置 `LLM_PRIMARY_*` / `LLM_DEEPSEEK_*`。其中 `LLM_<NAME>_API_KEY` / `LLM_<NAME>_API_KEYS` 当前也仅从 repository secrets 导入；如果你把这些值放在 Variables，运行时不会生效。若使用自定义渠道名（如 `my_proxy`），GitHub Actions 还必须在 workflow `env:` 中显式新增对应的 `LLM_MY_PROXY_*` 映射；本地 `.env` 和 Docker 不受这个限制。


> **三层配置互斥准则**：YAML 优先级最高！只要配置了 YAML，**渠道模式** 和 **新手极简模式** 统统被忽略。系统优先级为：`YAML配置 > 渠道模式 > 极简单模型`。

---

## 扩展功能：看图模型 (Vision) 配置

系统中有些特定功能（比如上传股票软件截图，让 AI 提取出截图里的股票代码并放入自选股池）必须用到具备“视觉能力”的模型。你需在 `.env` 单独给它指派一个懂图片的模型。

```env
# 指定你看图专用的模型名
VISION_MODEL=gemini/gemini-2.5-flash
# 别忘了填写它对应提供商的 API KEY，如果是 gemini 就提供 GEMINI_API_KEY：
# GEMINI_API_KEY=xxx
```

**备用看图机制：** 为了防止偶尔罢工，系统内置了切换策略。如果主视觉模型调用失败，它会按照下方的顺位尝试寻找是否有其他看图模型的 Key：
```env
# 默认的备用顺序：
VISION_PROVIDER_PRIORITY=gemini,anthropic,openai
```

---

## 检测与排错 (Troubleshooting)

配好了之后心惊胆战不知道对不对？在命令行（Terminal）里敲入下面代码帮你挂号问诊：

- `python test_env.py --config` ：纯检测 `.env` 配置文件里的逻辑写得对不对，是不是少写了什么。（秒出结果，不调用网络，纯检查本地文本拼写）
- `python test_env.py --llm` ：系统会真的发一句问候语给大模型，让你亲眼看到他的回答。这能彻底测出你的**网络通不通、账号有没有欠费**。

### 常见踩坑答疑台

| 遇到了什么诡异报错？ | 罪魁祸首可能是啥？ | 该怎么收拾它？ |
|----------------------|----------------------|------------------|
| **界面提示主模型未配置** | 系统不知道你到底想用哪家的哪个模型 | 在 `.env` 中写上一句明白话：`LITELLM_MODEL=provider/你的模型名`。比如 `openai/gpt-4o-mini` |
| **我写了好几家的Key，为什么死活只有一个生效？修改还没用？** | 你把 **极简模式** 和 **渠道模式** 混着写了！ | 想好一条路走到黑——只要简单就删掉 `LLM_CHANNELS` 开头的；想要丰富备用切换就要全部转投到 `LLM_CHANNELS` 下的编制里。 |
| **错误码报 400 或 401 或 Invalid API Key** | API Key 填错、少复制了一截、账号充值没到账、或者模型名字敲错（极度常见）。 | 1. 检查复制的 Key 前后是否有误填空格。<br> 2. 检查 Base URL 最后是不是少了一个 `/v1`。<br> 3. 检查模型名是否少写了 `openai/` 之类的前缀！ |
| **Kimi K2.6 报 `invalid temperature`（可能提示只允许 `1.0` 或 `0.6`）** | 该模型按 thinking / non-thinking 模式要求不同固定 temperature；旧配置或调用入口可能还在传 `0.7`。 | 升级后系统会对 `kimi-k2.6` 默认 / thinking 请求自动使用 `temperature=1.0`；如果你在 LiteLLM YAML 路由里显式关闭 thinking，则自动改用 `0.6`。模型名建议写成 `openai/kimi-k2.6` 并配合 Moonshot / 聚合平台的 OpenAI 兼容 Base URL 与 API Key。非 Kimi fallback 仍会继续使用你配置的 `LLM_TEMPERATURE`。 |
| **转圈转不停，最后报 Timeout / ConnectionRefused 等** | 1. 在国内使用国外原版（像 Google、OpenAI），没开代理被墙了。<br>2. 你买的云服务器压根不能出境。 | 非常推荐使用**国内官方**（如DeepSeek、阿里）或者各种**兼容 OpenAI 的聚合中转接口**。因为中转站把网络问题帮你解决好了。 |
| **Ollama 报 404、`Could not get model info` 或 `api/generate/api/show`** | 误用 `OPENAI_BASE_URL` 配置 Ollama，系统会错误拼接 URL | 改用 `OLLAMA_API_BASE=http://localhost:11434` 或渠道模式（`LLM_CHANNELS=ollama` + `LLM_OLLAMA_BASE_URL`） |

*进阶老手的叮嘱：如果你开启了 **Agent (深度思考网络搜索问股) 模式**，这里有个经验之谈，推荐选用如 `deepseek-v4-pro` 这种逻辑推导能力更强的大模型。如果为了省钱用小微模型跑 Agent，它逻辑能力大概率跟不上，不仅达不到预期，还会白跑一堆空流程。*
