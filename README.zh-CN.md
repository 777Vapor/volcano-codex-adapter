# Volcano Codex Adapter

[English](README.md) | 简体中文

一组用于将本地项目切换到火山引擎方舟 OpenAI 兼容 Responses API 配置的工具，支持回滚，并可针对 Codex 风格的 `User-Agent` 路由执行冒烟测试。

## 文件说明

- `switch_to_ark.sh`：检测项目类型并应用方舟模型配置。
- `smoke_test_ark_responses.sh`：向方舟发送兼容 Codex 的 Responses API 测试请求。
- `tests/run_tests.sh`：针对应用配置、状态检查、回滚和试运行行为的本地 Shell 测试。

## 支持的项目类型

- Codex 单体仓库：通过 `codex-rs/Cargo.toml` 和 `codex-cli/package.json` 检测。
- 兼容 OpenAI 的 Node 项目：通过 `package.json` 中的 `openai` 或 `@ai-sdk/openai` 依赖检测。
- 兼容 OpenAI 的 Python 项目：通过包含 `openai` 的 `requirements.txt` 或 `pyproject.toml` 检测。
- 未知项目：默认写入兼容 OpenAI 的 `.env` 环境变量。

## 将项目切换到方舟

对于 Codex 项目，脚本会写入项目本地的 `.codex/config.toml`。配置通过环境变量引用 `ARK_API_KEY`，不会将密钥写入 TOML 文件。

```bash
cd /path/to/project
export ARK_API_KEY="..."
export ARK_MODEL="ep-..."
/path/to/volcano-codex-adapter/switch_to_ark.sh apply
```

随后使用生成的配置文件运行 Codex：

```bash
ARK_API_KEY="..." codex -p ark
```

对于 Node、Python 和未知类型的项目，脚本会将兼容 OpenAI 的变量写入 `.env`：

```text
ARK_API_KEY=...
ARK_MODEL=...
ARK_BASE_URL=https://ark.cn-beijing.volces.com/api/v3
OPENAI_API_KEY=...
OPENAI_BASE_URL=https://ark.cn-beijing.volces.com/api/v3
OPENAI_MODEL=...
OPENAI_USER_AGENT=...
```

只有当目标应用会读取 `OPENAI_USER_AGENT`，并将其作为自定义请求头传递给 SDK 时，该变量才会生效。

## 回滚

每次执行 `apply` 都会在项目内的以下目录创建备份：

```text
.ark-switch/backups/<timestamp>/
```

回滚最近一次改动：

```bash
/path/to/volcano-codex-adapter/switch_to_ark.sh rollback
```

回滚到指定备份：

```bash
/path/to/volcano-codex-adapter/switch_to_ark.sh rollback --backup 20260730-113804
```

检查当前状态：

```bash
/path/to/volcano-codex-adapter/switch_to_ark.sh status
```

只预览操作、不写入文件：

```bash
ARK_API_KEY="..." ARK_MODEL="ep-..." \
  /path/to/volcano-codex-adapter/switch_to_ark.sh apply --dry-run
```

## 方舟 Responses API 冒烟测试

使用该脚本验证方舟是否接受 Codex 风格的 Responses API 请求和工具 schema。

```bash
export ARK_API_KEY="..."
export ARK_MODEL="ep-..."
./smoke_test_ark_responses.sh
```

可选环境变量：

```bash
export ARK_RESPONSES_URL="https://ark.cn-beijing.volces.com/api/v3/responses"
export USER_AGENT="codex_exec/0.0.0 (Mac OS; arm64) dumb (codex_exec; 0.0.0)"
export MAX_TIME=60
```

如果未设置 `ARK_API_KEY`，脚本可以从 `LLM_ENV_PATH` 指定的文件中读取：

```bash
export LLM_ENV_PATH="/Users/bytedance/WorkSpace/LLM_env.md"
./smoke_test_ark_responses.sh
```

## 本地测试

运行：

```bash
./tests/run_tests.sh
```

测试会创建临时测试数据，不会访问网络。

## 注意事项

- 默认作用域仅限当前项目。脚本不会修改 `~/.codex/config.toml`。
- 即使请求设置了 `reasoning.summary = "none"`，方舟仍可能发出 `response.reasoning_summary_*` SSE 事件。
- Codex provider 配置会通过 `http_headers` 注入 Codex 风格的 `User-Agent`，方舟的兼容路由可能需要该请求头。
