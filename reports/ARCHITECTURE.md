# New API 项目框架文档

> 基于源码分析自动生成 | 分析日期: 2026-04-25

---

## 1. 项目概述

**New API** 是一个 AI API 网关/代理管理平台，核心功能是将多种 AI 大模型供应商的 API 统一为 OpenAI 兼容格式对外提供服务，并提供完整的用户管理、计费、监控等运营能力。

| 属性 | 值 |
|------|-----|
| 项目名 | QuantumNous/new-api |
| 语言 | Go 1.25.1 (后端) + React 18 (前端) |
| Web 框架 | Gin |
| ORM | GORM |
| 数据库 | SQLite(默认) / MySQL / PostgreSQL |
| 缓存 | Redis (可选) + 内存缓存 |
| 前端构建 | Vite 5 + TailwindCSS 3 |
| UI 组件库 | Semi UI |
| 国际化 | i18next (前端7语言) + 自研i18n (后端3语言) |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端 (React SPA)                        │
│  Semi UI + TailwindCSS + i18next + VChart + SSE                 │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTP/SSE/WebSocket
┌────────────────────────────▼────────────────────────────────────┐
│                     Gin HTTP Server                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐     │
│  │  Router   │→│Middleware │→│Controller │→│   Service     │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────┬───────┘     │
│                                                      │          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────▼───────┐     │
│  │  Model   │  │  Setting  │  │  Common   │  │    Relay     │    │
│  │  (ORM)   │  │  (Config) │  │  (Utils)  │  │  (Adaptors)  │    │
│  └────┬─────┘  └──────────┘  └──────────┘  └──────┬───────┘    │
│       │                                             │           │
└───────┼─────────────────────────────────────────────┼───────────┘
        │                                             │
┌───────▼───────┐                         ┌───────────▼───────────┐
│   Database     │                         │  上游 AI 供应商 API    │
│ SQLite/MySQL/  │                         │ OpenAI/Claude/Gemini/ │
│ PostgreSQL     │                         │ AWS/阿里/百度/...     │
└───────────────┘                         └───────────────────────┘
```

### 2.2 请求处理主流程

```
用户请求
  → Router (路由匹配)
  → Middleware (认证/限流/分发)
    → RequestId → I18n → CORS → Gzip → RateLimit
    → TokenAuth/UserAuth → ModelRateLimit → Distribute
  → Controller (请求处理)
    → 敏感词检测 → Token估算 → 价格计算 → 预扣费
    → 渠道选择(亲和性优先→随机负载均衡)
    → 重试循环(最多N次)
  → Relay (供应商适配)
    → Adaptor选择(按渠道类型)
    → 请求格式转换(OpenAI→供应商格式)
    → HTTP请求转发
    → 响应格式转换(供应商格式→OpenAI)
    → 流式/非流式处理
  → Service (后结算)
    → 实际消耗计算 → 结算(预扣费调整) → 日志记录 → 额度通知
```

---

## 3. 目录结构

```
new-api/
├── main.go                 # 程序入口
├── go.mod / go.sum         # Go 依赖
├── Dockerfile              # 容器化
├── docker-compose.yml      # 编排配置
├── makefile                # 构建脚本
├── VERSION                 # 版本号
│
├── router/                 #   路由定义 (6文件)
│   ├── main.go             #   路由主入口 SetRouter
│   ├── api-router.go       #   /api/* 管理API路由
│   ├── relay-router.go     #   /v1/* AI模型转发路由
│   ├── web-router.go       #   前端静态资源路由
│   ├── video-router.go     #   视频相关路由
│   └── dashboard.go        #   旧版billing兼容路由
│
├── middleware/             #   HTTP中间件 (21文件)
│   ├── auth.go             #   认证 (Session/Token/APIKey)
│   ├── distributor.go      #   请求分发 (渠道选择)
│   ├── rate-limit.go       #   全局限流 (IP滑动窗口)
│   ├── model-rate-limit.go #   模型级限流 (令牌桶+滑动窗口)
│   ├── cors.go             #   跨域
│   ├── i18n.go             #   国际化
│   ├── logger.go           #   请求日志
│   ├── cache.go            #   缓存头
│   ├── performance.go      #   系统负载检查
│   ├── turnstile-check.go  #   Cloudflare人机验证
│   ├── secure_verification.go # 安全验证
│   └── ...                 #   其他辅助中间件
│
├── controller/              # 控制器 (61文件)
│   ├── relay.go            #   AI模型转发入口
│   ├── channel.go          #   渠道管理CRUD
│   ├── token.go            #   令牌管理CRUD
│   ├── user.go             #   用户管理/登录/注册
│   ├── log.go              #   日志查询
│   ├── billing.go          #   计费查询(兼容OpenAI)
│   ├── option.go           #   系统选项管理
│   ├── topup.go            #   充值(Epay/Stripe/Creem/Waffo)
│   ├── subscription.go     #   订阅管理
│   ├── redemption.go       #   兑换码管理
│   ├── midjourney.go       #   Midjourney任务
│   ├── task.go             #   异步任务
│   ├── model.go            #   模型列表
│   ├── oauth.go            #   OAuth登录
│   ├── playground.go       #   API Playground
│   ├── pricing.go          #   定价信息
│   ├── setup.go            #   系统初始化
│   └── ...                 #   其他控制器
│
├── service/                 # 业务逻辑服务 (55文件)
│   ├── billing.go          #   计费入口(预扣费/结算)
│   ├── billing_session.go  #   计费会话生命周期
│   ├── quota.go            #   配额计算与消费
│   ├── text_quota.go       #   文本配额详细计算
│   ├── channel_select.go   #   渠道选择(分组/跨组重试)
│   ├── channel_affinity.go #   渠道亲和性(会话保持)
│   ├── token_counter.go    #   Token计数
│   ├── sensitive.go        #   敏感词检测(AC自动机)
│   ├── midjourney.go       #   Midjourney服务
│   ├── task.go             #   异步任务服务
│   ├── convert.go          #   格式转换(Claude→OpenAI)
│   ├── http.go             #   HTTP请求工具
│   ├── webhook.go          #   Webhook通知
│   ├── notify-limit.go     #   通知频率限制
│   ├── openaicompat/       #   OpenAI Chat/Responses兼容转换
│   └── passkey/            #   Passkey/WebAuthn服务
│
├── model/                   # 数据模型 (36文件)
│   ├── main.go             #   数据库初始化/迁移
│   ├── channel.go          #   Channel渠道模型
│   ├── token.go            #   Token令牌模型
│   ├── user.go             #   User用户模型
│   ├── log.go              #   Log日志模型
│   ├── ability.go          #   Ability模型能力(联合主键)
│   ├── option.go           #   Option系统选项(KV存储)
│   ├── pricing.go          #   Pricing定价(内存计算)
│   ├── subscription.go     #   订阅(Plan/Order/UserSubscription/PreConsume)
│   ├── topup.go            #   TopUp充值模型
│   ├── redemption.go       #   Redemption兑换码模型
│   ├── midjourney.go       #   Midjourney任务模型
│   ├── task.go             #   Task异步任务模型
│   ├── channel_cache.go    #   渠道内存缓存
│   ├── token_cache.go      #   令牌Redis缓存
│   ├── user_cache.go       #   用户Redis缓存
│   └── ...                 #   其他模型
│
├── relay/                   # AI模型转发适配层 (192文件, 核心模块)
│   ├── relay_adaptor.go    #   适配器入口(24种同步+9种异步)
│   ├── relay_task.go       #   异步任务转发
│   ├── compatible_handler.go # 同步文本转发核心
│   ├── chat_completions_via_responses.go # Chat→Responses桥接
│   ├── common/             #   通用逻辑(RelayInfo/Billing/Override/Stream)
│   ├── constant/           #   转发模式常量(31种)
│   ├── helper/             #   辅助函数(SSE/模型映射/价格计算/流扫描)
│   ├── channel/            #   供应商适配器
│   │   ├── openai/         #     OpenAI (基准实现)
│   │   ├── claude/         #     Anthropic Claude
│   │   ├── gemini/         #     Google Gemini
│   │   ├── aws/            #     AWS Bedrock
│   │   ├── vertex/         #     Google Vertex AI
│   │   ├── ali/            #     阿里云通义千问
│   │   ├── baidu/          #     百度文心一言
│   │   ├── tencent/        #     腾讯混元
│   │   ├── zhipu/          #     智谱AI
│   │   ├── deepseek/       #     DeepSeek
│   │   ├── moonshot/       #     月之暗面
│   │   ├── minimax/        #     MiniMax
│   │   ├── mistral/        #     Mistral AI
│   │   ├── cohere/         #     Cohere
│   │   ├── xai/            #     xAI (Grok)
│   │   ├── volcengine/     #     火山引擎(豆包)
│   │   ├── xunfei/         #     讯飞星火
│   │   ├── ollama/         #     Ollama(本地)
│   │   ├── dify/           #     Dify
│   │   ├── coze/           #     Coze
│   │   ├── cloudflare/     #     Cloudflare Workers AI
│   │   ├── siliconflow/    #     SiliconFlow
│   │   ├── openrouter/     #     OpenRouter
│   │   ├── codex/          #     Codex
│   │   └── ...             #     其他(30+供应商)
│   └── channel/task/       #   异步任务适配器
│       ├── ali/            #     阿里
│       ├── doubao/         #     豆包
│       ├── gemini/         #     Gemini
│       ├── hailuo/         #     海螺
│       ├── jimeng/         #     即梦
│       ├── kling/          #     可灵
│       ├── sora/           #     Sora
│       ├── suno/           #     Suno
│       ├── vertex/         #     Vertex
│       └── vidu/           #     Vidu
│
├── dto/                     # 数据传输对象 (29文件)
│   ├── openai_request.go   #   OpenAI请求格式
│   ├── openai_response.go  #   OpenAI响应格式
│   ├── claude.go           #   Claude格式DTO
│   ├── gemini.go           #   Gemini格式DTO
│   └── ...                 #   其他DTO
│
├── common/                  # 通用工具 (46文件)
│   ├── constants.go        #   全局常量/开关变量
│   ├── env.go              #   环境变量工具
│   ├── init.go             #   命令行参数+环境变量初始化
│   ├── database.go         #   数据库类型常量
│   ├── redis.go            #   Redis连接
│   ├── email.go            #   邮件发送
│   ├── crypto.go           #   加密工具
│   ├── rate-limit.go       #   限流器
│   ├── limiter/            #   Lua限流脚本
│   └── ...                 #   其他工具
│
├── constant/                # 常量定义 (14文件)
│   ├── api_type.go         #   API类型常量
│   ├── channel.go          #   渠道类型常量
│   └── ...                 #   其他常量
│
├── setting/                 # 配置管理 (44文件)
│   ├── config/             #   全局配置管理器(ConfigManager)
│   ├── ratio_setting/      #   倍率/价格配置
│   ├── operation_setting/  #   运营配置(支付/监控/签到等)
│   ├── model_setting/      #   模型特定配置(Claude/Gemini/Grok/Qwen)
│   ├── system_setting/     #   系统配置(OIDC/Passkey/Legal/Fetch)
│   ├── performance_setting/#   性能配置(磁盘缓存/监控阈值)
│   └── ...                 #   其他配置
│
├── oauth/                   # OAuth提供商 (8文件)
│   ├── provider.go         #   Provider接口定义
│   ├── registry.go         #   提供商注册表
│   ├── github.go           #   GitHub
│   ├── discord.go          #   Discord
│   ├── oidc.go             #   OIDC通用
│   ├── linuxdo.go          #   LinuxDO
│   └── generic.go          #   通用OAuth
│
├── i18n/                    # 后端国际化 (5文件)
│   ├── i18n.go             #   初始化
│   ├── keys.go             #   翻译键
│   └── locales/            #   语言包(en/zh-CN/zh-TW)
│
├── logger/                  # 日志模块 (1文件)
├── types/                   # 通用类型 (9文件)
├── pkg/                     # 独立包
│   ├── cachex/             #   混合缓存(内存+Redis)
│   └── ionet/              #   io.net GPU集群集成
│
├── web/                     # 前端React应用 (415文件)
│   ├── src/
│   │   ├── App.jsx         #   路由配置
│   │   ├── context/        #   状态管理(Status/User/Theme)
│   │   ├── helpers/        #   工具函数(API/认证/渲染/配额)
│   │   ├── pages/          #   页面组件(25+页面)
│   │   ├── components/     #   UI组件
│   │   │   ├── layout/     #     布局(Header/Sider/Footer)
│   │   │   ├── table/      #     数据表格(渠道/令牌/用户/日志等)
│   │   │   ├── auth/       #     认证(登录/注册/OAuth/2FA)
│   │   │   ├── playground/ #     API Playground
│   │   │   ├── topup/      #     充值/订阅
│   │   │   ├── dashboard/  #     仪表盘
│   │   │   ├── settings/   #     设置(系统/运营/倍率/模型/支付等)
│   │   │   └── common/     #     通用(ErrorBoundary/Markdown/Logo)
│   │   └── i18n/           #   前端国际化(7语言)
│   └── ...                 #   构建配置
│
└── docs/                    # 项目文档
```

---

## 4. 核心模块详解

### 4.1 启动流程

```
main.go → InitResources()
  1. godotenv.Load(".env")                    # 加载.env文件
  2. common.InitEnv()                         # 解析命令行参数+环境变量→全局变量
  3. logger.SetupLogger()                     # 日志系统
  4. ratio_setting.InitRatioSettings()        # 模型倍率默认值→RWMap
  5. service.InitHttpClient()                 # HTTP客户端
  6. service.InitTokenEncoders()              # Token编码器
  7. model.InitDB()                           # 数据库连接+迁移(支持SQLite/MySQL/PostgreSQL)
  8. model.CheckSetup()                       # 系统初始化检查
  9. model.InitOptionMap()                    # 数据库options→内存(传统+分层配置)
 10. model.GetPricing()                       # 模型定价初始化
 11. model.InitLogDB()                        # 日志数据库
 12. common.InitRedisClient()                 # Redis连接
 13. common.StartSystemMonitor()              # 系统监控
 14. i18n.Init()                              # 国际化
 15. oauth.LoadCustomProviders()              # 自定义OAuth

main() 后台任务:
  - model.SyncChannelCache()                  # 渠道缓存定时同步
  - model.SyncOptions()                       # 配置热更新
  - model.UpdateQuotaData()                   # 数据看板
  - controller.AutomaticallyTestChannels()    # 自动测试渠道
  - service.StartCodexCredentialAutoRefreshTask()  # Codex凭证刷新
  - service.StartSubscriptionQuotaResetTask() # 订阅额度重置
  - controller.UpdateMidjourneyTaskBulk()     # MJ任务批量更新
  - controller.UpdateTaskBulk()               # 异步任务批量更新
```

### 4.2 配置体系

**双层配置架构**:

| 层级 | 存储位置 | Key格式 | 示例 | 管理方式 |
|------|----------|---------|------|----------|
| 传统平面配置 | `common.OptionMap` | 简单字符串 | `SystemName`, `SMTPServer` | `updateOptionMap()` switch分发 |
| 分层配置 | `config.GlobalConfig` | 模块名.字段名 | `general_setting.quota_display_type` | `ConfigManager.LoadFromDB()` |

**配置来源优先级**: 环境变量(启动时) → 数据库options表(支持热更新) → 代码默认值

**ConfigManager 核心方法**:
- `Register(name, config)` — init()阶段注册配置模块
- `LoadFromDB(options)` — 从数据库加载
- `SaveToDB(updateFunc)` — 保存到数据库
- `ExportAllConfigs()` — 导出所有配置为扁平结构

### 4.3 认证体系

| 认证方式 | 中间件 | 适用场景 | 验证逻辑 |
|----------|--------|----------|----------|
| Session | `UserAuth/AdminAuth/RootAuth` | Web管理后台 | Cookie Session → AccessToken Header |
| Token | `TokenAuth` | API调用 | Authorization Header → sk-xxx → model.ValidateUserToken |
| Token+Session | `TokenOrUserAuth` | 视频代理 | 先Session后Token回退 |
| ReadOnly Token | `TokenAuthReadOnly` | 用量查询 | 仅验证Key存在，不检查状态/额度 |

**Token认证特殊处理**:
- WebSocket: 从 `Sec-WebSocket-Protocol` 提取Key
- Claude: 从 `x-api-key` Header提取Key
- Gemini: 从 `key` query参数或 `x-goog-api-key` Header提取Key
- 管理员指定渠道: Key中"-"后部分作为specific_channel_id

### 4.4 渠道选择与负载均衡

```
Distribute中间件:
  1. 管理员指定渠道? → 直接使用
  2. 渠道亲和性匹配? → 使用亲和渠道(会话保持)
  3. 随机选择 → GetRandomSatisfiedChannel(group, model, retry)
     → 按优先级分层 → 同优先级按权重加权随机
  4. 跨分组重试 → autoGroups遍历 / CrossGroupRetry
```

**渠道亲和性**: 基于规则匹配(ModelRegex/PathRegex/UserAgent/KeySource) → HybridCache(Redis+LRU) → 缓存channelID。成功请求后记录亲和性，后续请求优先使用同一渠道。

### 4.5 计费流程

```
1. 预扣费 (PreConsumeBilling)
   ├── 创建BillingSession(根据计费偏好: subscription_first/wallet_first/...)
   ├── 信任额度旁路: 用户额度>trustQuota时跳过预扣
   ├── 令牌预扣: DecreaseTokenQuota
   └── 资金来源预扣: 钱包DecreaseUserQuota / 订阅PreConsumeUserSubscription

2. 请求转发 (Relay)
   └── 失败时: Refund退还预扣费

3. 后结算 (SettleBilling / PostTextConsumeQuota)
   ├── 计算实际消耗quota
   │   文本: (prompt + completion*completionRatio + cache*cacheRatio + ...) * modelRatio * groupRatio
   │   音频: (textIn + textOut*completionRatio + audioIn*audioRatio + ...) * modelRatio * groupRatio
   ├── delta = actualQuota - preConsumedQuota
   ├── BillingSession.Settle: 调整资金来源 + 调整令牌额度
   ├── 更新用户/渠道已用额度
   ├── 记录消费日志
   └── 额度不足通知(Email/Webhook/Bark/Gotify)
```

### 4.6 Relay转发核心

**适配器接口体系**:
```go
type Adaptor interface {
    Init(*common.RelayInfo)
    GetRequestURL() string
    SetupRequestHeader(c *gin.Context, req *http.Request)
    ConvertRequest() (any, error)
    DoRequest() (*http.Response, error)
    DoResponse() (usage *dto.Usage, err *types.NewAPIError)
}
```

**31种转发模式** (relay_mode):
- ChatCompletions, Completions, Embeddings, ImagesGenerations, AudioSpeech, AudioTranscription, Rerank, Responses, Realtime...
- ClaudeMessages, GeminiChat, GeminiEmbedding...
- Midjourney相关, Suno相关, Task相关

**流式处理核心** (StreamScannerHandler):
- 三goroutine架构: 读取goroutine → 扫描goroutine → 写入goroutine
- 支持SSE/JSON流式/WebSocket
- 自动检测流式错误并处理

**请求转换链** (RequestConversionChain):
- 支持多步转换: OpenAI → Claude → AWS Bedrock 等
- 每步转换可独立处理参数映射/格式差异

---

## 5. 数据库设计

### 5.1 核心表结构

| 表名 | 主键 | 核心字段 | 说明 |
|------|------|----------|------|
| `channels` | id | type, key, base_url, models, group, status, weight, priority | 渠道(上游API供应商) |
| `tokens` | id | user_id, key, status, remain_quota, group, model_limits | API令牌 |
| `users` | id | username, password, role, status, email, quota, group | 用户 |
| `abilities` | (group,model,channel_id) | enabled, priority, weight | 模型能力(渠道×模型×分组) |
| `logs` | id | user_id, model_name, quota, prompt_tokens, completion_tokens, channel_id | 请求日志 |
| `options` | key | value | 系统配置KV存储 |
| `top_ups` | id | user_id, amount, money, trade_no, payment_method, status | 充值订单 |
| `redemptions` | id | key, quota, status, expired_time | 兑换码 |
| `tasks` | id | task_id, platform, user_id, channel_id, action, status, properties | 异步任务 |
| `midjourneys` | id | user_id, mj_id, action, status, progress, image_url | Midjourney任务 |
| `subscription_plans` | id | title, price_amount, currency, duration_unit/value, total_amount | 订阅套餐 |
| `subscription_orders` | id | user_id, plan_id, trade_no, payment_method, status | 订阅订单 |
| `user_subscriptions` | id | user_id, plan_id, amount_total/used, start/end_time, status | 用户订阅实例 |
| `subscription_pre_consume_records` | id | request_id, user_subscription_id, pre_consumed, status | 订阅预消费记录 |

### 5.2 表关系

```
User (1) ──── (N) Token
User (1) ──── (N) Log
User (1) ──── (N) TopUp
User (1) ──── (N) Midjourney
User (1) ──── (N) Task
User (1) ──── (N) UserSubscription
User (1) ──── (N) SubscriptionOrder

Channel (1) ──── (N) Ability     [联合主键(group,model,channel_id)]
Channel (1) ──── (N) Log
Channel (1) ──── (N) Midjourney
Channel (1) ──── (N) Task

SubscriptionPlan (1) ──── (N) SubscriptionOrder
SubscriptionPlan (1) ──── (N) UserSubscription
UserSubscription (1) ──── (N) SubscriptionPreConsumeRecord
```

### 5.3 缓存策略

| 缓存对象 | 存储位置 | Key格式 | 更新策略 |
|----------|----------|---------|----------|
| 渠道 | 内存map | group→model→[]channelId | 定时全量同步(SyncFrequency) |
| 令牌 | Redis Hash | token:{HMAC(key)} | 写时更新+Redis原子操作 |
| 用户 | Redis Hash | user:{userId} | 写时更新+Redis原子操作 |
| 配置 | 内存map | common.OptionMap | 定时从DB同步(热更新) |
| 渠道亲和性 | Redis+LRU | affinity:{rule}:{value} | 成功请求后记录+TTL过期 |

---

## 6. API路由总览

### 6.1 管理API (`/api/*`)

| 路径 | 方法 | 认证 | 说明 |
|------|------|------|------|
| `/api/user/register` | POST | Turnstile | 用户注册 |
| `/api/user/login` | POST | Turnstile | 用户登录 |
| `/api/user/self/*` | * | UserAuth | 用户自身操作 |
| `/api/user/*` (管理) | * | AdminAuth | 用户管理 |
| `/api/channel/*` | * | AdminAuth | 渠道管理 |
| `/api/token/*` | * | UserAuth | 令牌管理 |
| `/api/log/*` | * | AdminAuth/UserAuth | 日志查询 |
| `/api/option/*` | * | RootAuth | 系统配置 |
| `/api/redemption/*` | * | AdminAuth | 兑换码管理 |
| `/api/subscription/*` | * | UserAuth | 订阅操作 |
| `/api/subscription/admin/*` | * | AdminAuth | 订阅管理 |
| `/api/models/*` | * | AdminAuth | 模型元数据管理 |
| `/api/pricing` | GET | TryUserAuth | 定价信息 |
| `/api/setup` | GET/POST | 无 | 系统初始化 |

### 6.2 AI模型转发 (`/v1/*`)

| 路径 | 方法 | 认证 | 说明 |
|------|------|------|------|
| `/v1/chat/completions` | POST | TokenAuth | Chat补全(核心) |
| `/v1/completions` | POST | TokenAuth | Text补全 |
| `/v1/embeddings` | POST | TokenAuth | 向量嵌入 |
| `/v1/images/generations` | POST | TokenAuth | 图像生成 |
| `/v1/audio/speech` | POST | TokenAuth | TTS语音合成 |
| `/v1/audio/transcriptions` | POST | TokenAuth | 语音识别 |
| `/v1/rerank` | POST | TokenAuth | 重排序 |
| `/v1/responses` | POST | TokenAuth | Responses API |
| `/v1/realtime` | WS | TokenAuth | WebSocket实时API |
| `/v1/messages` | POST | TokenAuth | Claude Messages |
| `/v1/models` | GET | TokenAuth | 模型列表 |
| `/v1beta/*` | * | TokenAuth | Gemini原生API |
| `/mj/*` | POST | TokenAuth | Midjourney |
| `/suno/*` | POST | TokenAuth | Suno音乐 |

---

## 7. 前端架构

### 7.1 技术栈

| 类别 | 技术 |
|------|------|
| 构建工具 | Vite 5 (手动分包优化) |
| 框架 | React 18.2 |
| 路由 | react-router-dom 6.3 |
| UI | Semi UI 2.69 + TailwindCSS 3 |
| HTTP | Axios 1.15 (GET请求自动去重) |
| 国际化 | i18next (7语言: en/zh-CN/zh-TW/fr/ja/ru/vi) |
| 图表 | VChart |
| SSE | sse.js 2.6 |
| 状态管理 | React Context + useReducer |

### 7.2 页面路由

| 路径 | 守卫 | 页面 |
|------|------|------|
| `/` | 无 | 首页 |
| `/login` | AuthRedirect | 登录 |
| `/register` | AuthRedirect | 注册 |
| `/console` | PrivateRoute | 仪表盘 |
| `/console/channel` | AdminRoute | 渠道管理 |
| `/console/token` | PrivateRoute | 令牌管理 |
| `/console/playground` | PrivateRoute | API Playground |
| `/console/topup` | PrivateRoute | 充值 |
| `/console/log` | PrivateRoute | 日志 |
| `/console/user` | AdminRoute | 用户管理 |
| `/console/setting` | AdminRoute | 系统设置 |
| `/console/redemption` | AdminRoute | 兑换码管理 |
| `/console/subscription` | AdminRoute | 订阅管理 |
| `/console/models` | AdminRoute | 模型管理 |
| `/console/task` | PrivateRoute | 任务日志 |
| `/pricing` | 可配置 | 模型定价 |

### 7.3 状态管理

| Context | 状态 | 说明 |
|---------|------|------|
| StatusContext | 系统状态/配置 | 从`/api/status`加载 |
| UserContext | 当前用户信息 | localStorage持久化 |
| ThemeContext | 主题(light/dark/auto) | 跟随系统主题 |

---

## 8. 支持的AI供应商

### 8.1 同步适配器 (24种)

| 供应商 | 渠道类型 | 支持能力 |
|--------|----------|----------|
| OpenAI | openai | Chat/Embedding/Image/Audio/Responses/Realtime/Modération |
| Azure OpenAI | azure | 同OpenAI |
| Anthropic Claude | anthropic_claude | Chat(Messages API) |
| Google Gemini | gemini | Chat/Embedding/Image(Imagen) |
| Google Vertex AI | vertex_ai | Claude/Gemini/OpenSource模型 |
| AWS Bedrock | bedrock | Claude/Nova(Converse API) |
| 阿里云通义千问 | ali | Chat/Embedding/Image |
| 百度文心一言 | baidu/baidu_v2 | Chat |
| 腾讯混元 | tencent | Chat |
| 智谱AI | zhipu/zhipu_4v | Chat/GLM-4V |
| DeepSeek | deepseek | Chat |
| 月之暗面 | moonshot | Chat |
| MiniMax | minimax | Chat |
| Mistral AI | mistral | Chat |
| Cohere | cohere | Chat/Rerank |
| xAI (Grok) | xai | Chat |
| 火山引擎(豆包) | volcengine | Chat |
| 讯飞星火 | xunfei | Chat |
| 360智脑 | ai360 | Chat |
| 零一万物 | lingyiwanwu | Chat |
| Cloudflare | cloudflare | Chat |
| SiliconFlow | siliconflow | Chat |
| Ollama | ollama | Chat(本地模型) |
| OpenRouter | openrouter | Chat(聚合路由) |

### 8.2 异步任务适配器 (10种)

| 平台 | 支持任务 |
|------|----------|
| 阿里 | 图像生成 |
| 豆包(doubao) | 图像/视频生成 |
| Gemini | 图像/视频生成 |
| 海螺(hailuo) | 视频生成 |
| 即梦(jimeng) | 图像生成 |
| 可灵(kling) | 视频/图像生成 |
| Sora | 视频生成 |
| Suno | 音乐生成 |
| Vertex | 图像/视频生成 |
| Vidu | 视频生成 |

---

## 9. 支付体系

| 支付方式 | 控制器 | 服务 | 说明 |
|----------|--------|------|------|
| 易支付(Epay) | topup.go | epay.go | 国内支付网关 |
| Stripe | topup_stripe.go | - | 国际信用卡 |
| Creem | topup_creem.go | - | 订阅支付 |
| Waffo | topup_waffo.go | - | 支付网关 |
| Waffo Pancake | topup_waffo_pancake.go | waffo_pancake.go | Waffo变体 |
| 兑换码 | user.go(TopUp) | - | 离线充值 |

---

## 10. 关键环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SQL_DSN` | 空(SQLite) | 数据库连接串(postgres:///mysql://) |
| `REDIS_CONN_STRING` | 空 | Redis连接串 |
| `SESSION_SECRET` | 随机UUID | 会话密钥 |
| `PORT` | 3000 | 服务端口 |
| `SYNC_FREQUENCY` | 60 | 缓存同步频率(秒) |
| `GLOBAL_API_RATE_LIMIT` | 180 | API限流数 |
| `MEMORY_CACHE_ENABLED` | false | 内存缓存开关 |
| `RELAY_TIMEOUT` | 0 | 转发超时(秒) |
| `FRONTEND_BASE_URL` | 空 | 外部前端地址(非主节点) |
| `CHANNEL_UPDATE_FREQUENCY` | 空 | 渠道自动更新频率 |

---

## 11. 中间件执行顺序

### API请求 (`/api/*`)
```
RequestId → I18n → GinRecovery → RouteTag("api") → Gzip
→ BodyStorageCleanup → GlobalAPIRateLimit → [认证] → [限流] → Controller
```

### Relay请求 (`/v1/*`)
```
RequestId → I18n → GinRecovery → CORS → DecompressRequest → BodyStorageCleanup
→ StatsMiddleware → RouteTag("relay") → SystemPerformanceCheck
→ TokenAuth → ModelRequestRateLimit → Distribute → Controller
```

### Web请求 (前端)
```
RequestId → I18n → GinRecovery → Gzip → GlobalWebRateLimit → Cache → StaticServe
```
