# copilot-api-fixed

> Fork from [ericc-ch/copilot-api](https://github.com/ericc-ch/copilot-api) (MIT License)

## 这个 Fork 解决了什么问题？

[OpenAI Codex CLI](https://github.com/openai/codex) 配置 `wire_api = "responses"` 时，会调用 `/v1/responses`（OpenAI Responses API）端点。而原版 [copilot-api](https://github.com/ericc-ch/copilot-api) **只实现了** `/v1/chat/completions` 和 `/v1/messages`，没有实现 `/v1/responses`。

这意味着：

- **gpt-5.4、gpt-5.3-codex** 等新模型，GitHub Copilot 后端**只允许**通过 Responses API 访问，拒绝 `/chat/completions` 请求（返回 `unsupported_api_for_model`）
- 原版 copilot-api 没有 `/v1/responses` 端点 → Codex CLI 请求直接 404
- 即使把 Codex CLI 配成 `wire_api = "chat"`，gpt-5.4 也会被上游拒绝

**结果：通过原版 copilot-api 反代，Codex CLI 无法使用 gpt-5.4。**

## 怎么解决的？

新增了 `/v1/responses` 和 `/responses` 路由，**直接透传到 GitHub Copilot 后端的 `/responses` 端点**：

```
Codex CLI  ──(Responses API)──>  copilot-api-fixed  ──(/responses)──>  GitHub Copilot 后端
```

不做格式转换，请求和响应直接透传，流式 SSE 事件也原样转发。

### 改动范围

| 文件 | 说明 |
|------|------|
| `src/services/copilot/create-responses.ts` | 新增：调用 Copilot 后端 `/responses` 端点 |
| `src/routes/responses/handler.ts` | 新增：处理请求，透传响应（含流式 SSE） |
| `src/routes/responses/route.ts` | 新增：注册 POST 路由 |
| `src/server.ts` | 修改：注册 `/v1/responses` 和 `/responses` 路由 |

## 安装方式

### 方式一：Homebrew（推荐，macOS Apple Silicon）

```bash
brew tap sq-lai/tap
brew install copilot-api
```

安装完成后：

```bash
copilot-api auth     # 首次认证 GitHub Copilot
copilot-api start    # 启动服务器，默认端口 4141
```

### 方式二：从源码构建

需要先安装 [Bun](https://bun.sh)（`curl -fsSL https://bun.sh/install | bash`）

```bash
git clone https://github.com/sq-lai/copilot-api-fixed.git
cd copilot-api-fixed
bun install
bun run src/main.ts auth    # 首次认证
bun run src/main.ts start   # 启动，默认端口 4141
```

### 方式三：下载预编译二进制

从 [Releases](https://github.com/sq-lai/copilot-api-fixed/releases) 页面下载对应平台的二进制文件：

```bash
# macOS Apple Silicon (arm64)
tar -xzf copilot-api-darwin-arm64.tar.gz
chmod +x copilot-api
./copilot-api auth    # 首次认证
./copilot-api start   # 启动
```

## 配置 Codex CLI

编辑 `~/.codex/config.toml`：

```toml
model_provider = "custom"
model = "gpt-5.4"
model_reasoning_effort = "high"
disable_response_storage = true

[model_providers]

[model_providers.custom]
name = "custom"
wire_api = "responses"
requires_openai_auth = false
base_url = "http://localhost:4141"
```

然后直接使用：

```bash
codex "你好"
```

## 支持的模型

通过 `/v1/responses` 端点可以使用包括但不限于：

- **gpt-5.4** (OpenAI)
- **gpt-5.3-codex** (OpenAI)
- **gpt-5.2-codex** (OpenAI)
- **claude-sonnet-4.6** (Anthropic)
- **gemini-3.1-pro-preview** (Google)

所有模型列表可通过 `curl http://localhost:4141/v1/models` 查看。

## 致谢

- [ericc-ch/copilot-api](https://github.com/ericc-ch/copilot-api) — 原始项目，MIT 协议

## License

MIT License — 见 [LICENSE](./LICENSE) 文件
