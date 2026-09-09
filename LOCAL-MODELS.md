# 本地模型接入指南（Local Fork）

本分叉在官方 Warp 源码基础上做了以下改造，使其能够接入本地/自建大模型（Ollama、LM Studio、llama.cpp server、vLLM 等 OpenAI/Anthropic 兼容端点）：

## 改动内容

| 改动 | 位置 |
| --- | --- |
| 自定义推理端点允许 `http://`、`localhost`、127.0.0.1、内网/私网 IP | `crates/ai/src/api_keys.rs` (`validate_custom_endpoint_url`) |
| 自定义端点 API Key 可留空（本地服务通常无需鉴权） | 同上 + `app/src/settings_view/custom_inference_modal.rs` |
| 无 Key 端点的模型正常出现在模型选择器中 | `app/src/ai/llms.rs` (`build_custom_llm_infos`) |
| 无 Key 端点正常随请求下发 | `crates/ai/src/api_keys.rs` (`custom_model_providers_for_request`) |
| `settings.toml` 中手写的 localhost 端点不再导致整组配置失效 | 反序列化校验复用上述函数 |

## 路径一：本地 CLI Harness（推荐，完全本地执行）

Warp 的 AI 代理可以在本地拉起第三方 CLI agent（Claude Code、Codex 等），这些 CLI
在**本机**执行，可以直接访问 `localhost` 上的模型服务。

### Codex Harness + Ollama / LM Studio（OpenAI 兼容）

1. 在服务器上安装 [Codex CLI](https://github.com/openai/codex) 和 [Ollama](https://ollama.com)。
2. 在 Warp 设置 → Agents → Harness 中选择 Codex，认证方式选 "OpenAI API Key"：
   - `OPENAI_API_KEY`：填任意非空值（如 `ollama`）
   - `BASE_URL`：填 `http://localhost:11434/v1`
3. Codex 会把所有推理请求发往本地的 Ollama。

### Claude Code Harness + 本地 Anthropic 兼容网关

Claude Code CLI 原生支持 `ANTHROPIC_BASE_URL` 环境变量。harness 子进程会继承
Warp 进程的环境变量，因此只需在启动 Warp 前设置好环境变量：

```powershell
# Windows（系统级，对 Windows Server 推荐）
setx /M ANTHROPIC_BASE_URL "http://localhost:4000"   # 例如 LiteLLM 代理
setx /M ANTHROPIC_AUTH_TOKEN "anything-non-empty"

# 然后重启 Warp（或重新登录）使设置生效
```

## 路径二：自定义推理端点（Warp 默认引擎）

设置 → Agents → Custom Inference Endpoint，新增端点：

- **Endpoint URL**：如 `http://localhost:11434/v1`（本分叉已允许 http 与本地地址）
- **API Key**：可留空
- **Schema**：`OpenAI Chat Completions`（Ollama/LM Studio/vLLM 均兼容）
- **Models**：手动填写模型名，如 `qwen2.5-coder:32b`

> ⚠️ 架构限制：Warp 默认引擎（Oz）的推理请求由 **Warp 云端服务器**代为拨号自定义端点。
> 因此该路径要求端点能被公网访问（例如服务器有公网 IP 时通过 nginx/caddy 暴露 HTTPS，
> 或使用隧道）。纯 `localhost` 的模型请使用路径一。
>
> 此外，默认引擎的每次请求都需要登录 Warp 账号（免费账号即可）。

也可直接编辑 `settings.toml`：

```toml
[agents.custom_endpoints.endpoint-00000000-0000-0000-0000-000000000001]
name = "Ollama"
base_url = "http://localhost:11434/v1"
schema = "openai_chat_completions"

[[agents.custom_endpoints.endpoint-00000000-0000-0000-0000-000000000001.models]]
name = "qwen2.5-coder:32b"
alias = "Qwen Coder 32B"
config_key = "00000000-0000-0000-0000-000000000002"
```

（`config_key` 为任意不重复的 UUID；API Key 通过 UI 保存到系统凭据库。）

## Windows 构建说明

- 产物：`warp.exe`（GUI 版）与 `warp-tui.exe`（终端 UI 版，适合无桌面的 Windows Server）。
- 构建方式：GitHub Actions `windows-latest` + MSVC 工具链，见
  `.github/workflows/build-windows.yml`。
- GUI 版需要桌面会话（Windows Server 需安装桌面体验）；纯命令行场景请用 TUI 版。
