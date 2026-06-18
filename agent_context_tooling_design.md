# PR-Agent 的 Agent 上下文管理与工具调用设计梳理

本文聚焦仓库中主要 agent 组件的设计：请求入口、上下文管理、工具路由、Git provider 适配、AI handler 调用、配置覆盖与一次 `/review` 的完整协作链路。

## 1. 总体架构

PR-Agent 的核心不是一个“自主规划型 agent”，而是一个命令驱动的 PR 工作流执行器。用户通过 CLI、Webhook、GitHub App、GitLab webhook 等入口触发命令，`PRAgent` 负责解析命令并分发给具体 tool class，每个 tool class 再通过统一的 Git provider 和 AI handler 完成外部平台读写与模型调用。

高层链路如下：

```text
CLI / Webhook / App server
        |
        v
PRAgent.handle_request()
        |
        +-- apply_repo_settings()
        +-- parse command and args
        +-- update settings from args
        +-- route to tool class
                |
                v
        PRReviewer / PRDescription / PRCodeSuggestions / ...
                |
                +-- GitProvider: read PR, files, diff, comments, labels
                +-- TokenHandler + pr_processing: build model-sized context
                +-- Prompt templates: render system/user prompts
                +-- BaseAiHandler implementation: call model
                +-- Output parser/formatter: YAML -> Markdown
                +-- GitProvider: publish comments, labels, descriptions
```

关键文件：

- `pr_agent/agent/pr_agent.py`: agent 命令入口和 tool 路由。
- `pr_agent/tools/pr_reviewer.py`: `/review` 主实现。
- `pr_agent/tools/pr_description.py`: `/describe` 主实现。
- `pr_agent/tools/pr_code_suggestions.py`: `/improve` 主实现。
- `pr_agent/git_providers/git_provider.py`: Git 平台抽象接口。
- `pr_agent/algo/ai_handlers/base_ai_handler.py`: 模型调用抽象接口。
- `pr_agent/algo/ai_handlers/litellm_ai_handler.py`: 默认 LiteLLM 实现。
- `pr_agent/config_loader.py`: 全局配置和请求上下文配置。
- `pr_agent/git_providers/utils.py`: repo-local / external 配置应用。

## 2. Agent 入口与命令路由

`PRAgent` 是主要 orchestrator，定义在 `pr_agent/agent/pr_agent.py`。

它通过 `command2class` 把命令映射到 tool class：

```python
command2class = {
    "auto_review": PRReviewer,
    "answer": PRReviewer,
    "review": PRReviewer,
    "review_pr": PRReviewer,
    "describe": PRDescription,
    "improve": PRCodeSuggestions,
    "ask": PRQuestions,
    ...
}
```

`PRAgent._handle_request()` 的职责主要有五类：

1. 应用仓库配置：调用 `apply_repo_settings(pr_url)`。
2. 解析请求：支持字符串命令和 list 形式命令。
3. 校验 CLI 参数：`CliArgs.validate_user_args()` 会拒绝通过 CLI 传入敏感配置，例如 token、secret、provider key。
4. 更新运行配置：`update_settings_from_args(args)` 将安全的 `--section.key=value` 参数写入 Dynaconf。
5. 分发到具体工具：比如 `/review` 会实例化 `PRReviewer(...).run()`。

这个设计使 agent 本身保持很薄：它不直接做 review、describe 或 improve，而是只做请求生命周期和工具调度。

## 3. 上下文管理设计

仓库里有三层上下文概念：配置上下文、Git provider 上下文、单个 tool 的 prompt 变量上下文。

### 3.1 配置上下文

配置入口在 `pr_agent/config_loader.py`。

默认配置通过 Dynaconf 加载，包括：

- `settings/configuration.toml`
- `settings/pr_reviewer_prompts.toml`
- `settings/pr_description_prompts.toml`
- `settings/code_suggestions/*.toml`
- `settings/ignore.toml`
- 其他 prompt 和配置文件

`get_settings()` 优先从 `starlette_context.context["settings"]` 获取请求级配置，如果当前不在 Web/App 请求上下文中，则回退到进程级 `global_settings`。

```python
def get_settings(use_context=False):
    try:
        return context["settings"]
    except Exception:
        return global_settings
```

这个设计兼容两种运行模式：

- CLI 模式：使用全局 `global_settings`。
- Server/Webhook 模式：可以使用 `starlette_context` 做请求隔离，避免并发请求互相污染配置。

### 3.2 配置覆盖顺序

`apply_repo_settings()` 负责把外部共享配置和仓库内 `.pr_agent.toml` 合并到当前配置中。

主要顺序是：

1. 默认配置文件。
2. 环境变量。
3. `--extra_config_url` 指向的外部共享配置。
4. Git provider 从仓库读取的 `.pr_agent.toml`。
5. 再次重放环境变量覆盖，确保环境变量优先级最高。
6. 命令行安全参数覆盖。

实现中有几个安全点：

- 禁止自动加载 `.env`。
- external config 限制 1 MB。
- 远程配置 URL 日志会去掉 query 和 userinfo，避免泄露凭据。
- repo settings 会经过 `custom_merge_loader` 和安全校验。
- CLI 参数禁止传入敏感 key，例如 `openai.key`、`personal_access_token`、`webhook_secret`。

### 3.3 Git provider 上下文缓存

`get_git_provider_with_context(pr_url)` 会优先尝试从 `starlette_context` 里复用 provider 实例；如果没有上下文或没有缓存，就根据 `config.git_provider` 实例化对应 provider。

这样做有两个目的：

- 同一请求内多个 tool/helper 可以复用同一个 provider 对象。
- Server 模式下可以把请求状态限制在当前 request context 中。

需要注意：当前代码里缓存读取使用了 `context.get("git_provider", {}).get("pr_url", {})`，而写入时是 `context["git_provider"] = {pr_url: git_provider}`。从意图看是按 PR URL 缓存 provider，但读取处使用了字符串 `"pr_url"` 作为 key，可能导致缓存命中逻辑不符合预期。

### 3.4 Tool 内部 prompt 变量上下文

每个 tool 都会构造自己的 `self.vars`，作为 prompt 渲染上下文。

以 `PRReviewer` 为例，它会收集：

- `title`: PR 标题。
- `branch`: PR 分支。
- `description`: PR 描述。
- `language`: 主语言。
- `diff`: 初始为空，后续填入压缩后的 diff。
- `num_pr_files`: 文件数量。
- `num_max_findings`: 最大问题数。
- `require_tests`、`require_security_review` 等 feature flags。
- `extra_instructions`: 用户额外指令。
- `commit_messages_str`: commit message。
- `related_tickets`: 关联需求或 ticket。
- `date`: 当前日期。

`TokenHandler` 会先用 `diff=""` 渲染 prompt，估算固定 prompt token 成本；随后 `get_pr_diff()` 根据剩余 token 预算裁剪 diff。

## 4. 工具调用设计

这里的“工具调用”有两层含义。

第一层是 PR-Agent 内部 tool class：

- `/review` -> `PRReviewer`
- `/describe` -> `PRDescription`
- `/improve` -> `PRCodeSuggestions`
- `/ask` -> `PRQuestions`
- `/ask_line` -> `PR_LineQuestions`
- `/generate_labels` -> `PRGenerateLabels`
- `/similar_issue` -> `PRSimilarIssue`

这些 tool 都遵循相近模式：

```text
__init__()
  - 获取 GitProvider
  - 获取 PR 元信息
  - 构造 vars
  - 初始化 AI handler
  - 初始化 TokenHandler

run()
  - 检查 PR 是否有文件
  - 发布临时进度评论
  - 构造 diff/context
  - 调用模型
  - 解析结构化输出
  - 发布结果
```

第二层是外部能力适配器：

- `GitProvider`: 调 GitHub/GitLab/Bitbucket/Azure/Gerrit/Gitea/Local。
- `BaseAiHandler`: 调模型 API。
- Secret provider: 可从 AWS Secrets Manager、GCP Secret Manager 等读取配置。

这种分层让 tool class 不依赖具体平台 SDK，也不依赖具体模型供应商。

## 5. Git Provider 抽象

`GitProvider` 是所有平台接入的统一接口，定义在 `pr_agent/git_providers/git_provider.py`。

核心能力包括：

- 读取 PR 文件：`get_files()`
- 读取 diff 文件：`get_diff_files()`
- 读取语言分布：`get_languages()`
- 读取 PR 分支：`get_pr_branch()`
- 读取/处理 PR 描述：`get_pr_description()`、`get_user_description()`
- 发布描述：`publish_description()`
- 发布普通评论：`publish_comment()`
- 发布持久评论：`publish_persistent_comment()`
- 发布代码建议：`publish_code_suggestions()`
- 发布 label：具体 provider 可实现 `publish_labels()`
- 生成行链接：`get_line_link()`
- 支持能力检测：`is_supported(capability)`

`get_git_provider()` 根据配置中的 `config.git_provider` 选择具体实现：

```python
_GIT_PROVIDERS = {
    "github": GithubProvider,
    "gitlab": GitLabProvider,
    "bitbucket": BitbucketProvider,
    "azure": AzureDevopsProvider,
    "local": LocalGitProvider,
    ...
}
```

这使得 `/review` 这种 tool 不需要关心 GitHub 和 GitLab 的 API 差异，只要依赖抽象方法即可。

## 6. AI Handler 抽象

模型调用抽象是 `BaseAiHandler`，定义了一个核心方法：

```python
async def chat_completion(
    self,
    model: str,
    system: str,
    user: str,
    temperature: float = 0.2,
    img_path: str = None,
)
```

默认实现是 `LiteLLMAIHandler`。它负责：

- 从配置读取 OpenAI、Anthropic、Bedrock、Vertex、OpenRouter、Gemini、Groq 等 key。
- 根据 provider 设置 LiteLLM 参数。
- 处理 Azure/OpenAI deployment。
- 处理只支持 user message 的模型，把 system/user 合并。
- 处理不支持 temperature 的模型。
- 支持 GPT-5 / reasoning model 的 `reasoning_effort`。
- 支持 Claude extended thinking。
- 对需要 streaming 的模型自动走 streaming。
- 接入 LiteLLM callbacks，如 Langfuse/LangSmith metadata。
- 做模型调用重试。

tool class 不直接调用 LiteLLM，而是只调用 `self.ai_handler.chat_completion()`，这给测试替换和自定义模型后端留出了接口。

## 7. Diff 与模型上下文构造

上下文构造是 agent 设计里最关键的一层，主要在 `pr_agent/algo/pr_processing.py` 和 `pr_agent/algo/token_handler.py`。

### 7.1 TokenHandler

`TokenHandler` 负责估算 token：

- 初始化时渲染 system/user prompt，计算固定 prompt token。
- `count_tokens()` 用 tiktoken 做估算。
- 对 Anthropic 可调用官方 token counting API 做更准确估算。
- 对未知模型可用配置中的估算因子放大 token 数。

### 7.2 get_pr_diff()

`get_pr_diff()` 的目标是把 PR diff 转成模型可消费的字符串，并控制在上下文窗口内。

流程：

1. 读取 provider 的 `get_diff_files()`。
2. 按主语言排序文件。
3. 给 patch 增加额外上下文行。
4. 把 hunk 转成带行号格式。
5. 计算总 token。
6. 如果没超限，返回完整扩展 diff。
7. 如果超限，进入 compressed diff：
   - 移除 delete-only hunk。
   - 按文件 diff token 降序处理。
   - 跳过单文件过大的 patch。
   - 在预算允许时补充未处理的 added/modified/deleted 文件列表。

`/review` 调用 `get_pr_diff(..., add_line_numbers_to_hunks=True)`，这样模型可以返回准确的 `start_line` 和 `end_line`。

## 8. Prompt 与结构化输出

评审 prompt 在 `pr_agent/settings/pr_reviewer_prompts.toml`。

它的设计不是让模型自由写一段 review，而是要求输出 YAML，并声明一个 Pydantic 风格 schema：

```text
PRReview
  review
    estimated_effort_to_review_[1-5]
    relevant_tests
    key_issues_to_review[]
      relevant_file
      issue_header
      issue_content
      start_line
      end_line
    security_concerns
```

配置项会决定 schema 中是否出现某些字段，例如：

- `require_score_review`
- `require_tests_review`
- `require_estimate_effort_to_review`
- `require_security_review`
- `require_todo_scan`
- `require_can_be_split_review`
- `require_ticket_analysis_review`

这种 prompt schema 有两个作用：

- 降低后处理复杂度，便于 YAML 解析。
- 把模型输出约束在产品需要的 review UI 结构里。

## 9. `/review` 的组件协作链路

一次 `/review` 大致如下：

```text
PRAgent._handle_request()
  |
  +-- apply_repo_settings(pr_url)
  +-- parse "review"
  +-- instantiate PRReviewer
        |
        +-- get_git_provider_with_context(pr_url)
        +-- get PR title/branch/description/files/languages
        +-- build self.vars
        +-- init TokenHandler
        |
        v
     PRReviewer.run()
        |
        +-- extract_and_cache_pr_tickets()
        +-- publish "Preparing review..."
        +-- retry_with_fallback_models(self._prepare_prediction)
              |
              +-- get_pr_diff()
              +-- render prompt
              +-- ai_handler.chat_completion()
        |
        +-- load_yaml()
        +-- convert_to_markdown_v2()
        +-- set_review_labels()
        +-- publish_persistent_comment() / publish_comment()
```

这里的 fallback 模型逻辑在 `retry_with_fallback_models()`：先用 `config.model`，失败后按 `config.fallback_models` 顺序尝试。

## 10. 增量评审

`PRReviewer` 支持 `-i` 参数进入 incremental review。

主要行为：

- 初始化时调用 provider 的 `get_incremental_commits()`。
- `_can_run_incremental_review()` 检查：
  - provider 是否支持增量。
  - 新 commit 数是否达到阈值。
  - 距离上次 commit 的时间是否达到阈值。
  - auto 模式下是否有新 commit。
- 如果没有未评审文件，会发布跳过说明。
- 输出 Markdown 时会标注从哪个 commit 开始评审。

增量能力依赖具体 Git provider 的实现，agent 只使用抽象字段和方法。

## 11. 输出后处理

模型输出不是直接发布，而是经过多步处理：

1. `load_yaml()` 解析 YAML，并对常见模型格式错误做修复。
2. `github_action_output()` 将结构化结果暴露给 GitHub Action 输出。
3. `convert_to_markdown_v2()` 转成 GitHub/GitLab 友好的 Markdown。
4. 对 `key_issues_to_review`：
   - 读取 `relevant_file`、`start_line`、`end_line`。
   - 调用 `git_provider.get_line_link()` 生成代码行链接。
   - 可抽取相关代码片段放入 details。
5. `set_review_labels()` 根据 effort 和 security 输出更新 label。
6. 根据配置选择 persistent comment 或普通 comment。

## 12. 设计特点

这个 agent 框架有几个明显特点：

1. 命令式而非开放式规划  
   agent 不做多步自主规划，而是由 `/review`、`/describe`、`/improve` 等命令触发确定性流程。

2. 上下文由代码严格构造  
   模型看到的上下文来自 PR 元信息、diff、ticket、用户指令和配置开关，而不是任意检索。

3. 外部平台通过 provider 解耦  
   GitHub/GitLab/Bitbucket 等差异被压在 `GitProvider` 子类中。

4. 模型通过 AI handler 解耦  
   工具层只面向 `BaseAiHandler`，默认实现交给 LiteLLM。

5. 输出结构化  
   prompt 强制 YAML schema，后处理再转 Markdown/comment/label。

6. 配置优先  
   大量行为通过 `.pr_agent.toml`、`settings/*.toml` 和环境变量控制，避免硬编码。

7. Token 预算是核心约束  
   `TokenHandler` 和 `get_pr_diff()` 共同决定哪些 diff 能进入模型上下文。

## 13. 可以继续关注的实现细节

如果后续要深入改造或排查问题，可以优先看这些点：

- `get_git_provider_with_context()` 的 provider 缓存 key 是否按预期命中。
- `get_pr_diff()` 超大 PR 时跳过大文件的策略是否会漏掉关键风险。
- `load_yaml()` 对模型异常输出的修复能力。
- `convert_to_markdown_v2()` 对不同 provider 的 Markdown 兼容。
- `LiteLLMAIHandler.chat_completion()` 中不同模型能力差异的参数适配。
- repo settings 和 env overrides 的优先级是否符合部署预期。
