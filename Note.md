###这个分支主要是为了实现opencode的上下文管理改造成分层流水线模型，然后在分层之间加上hooks的拦截层，为的是实现针对不同大模型的上下文个性化的扩展。想法来源是Reasonix对deepseek的专项优化。
但是Reasonix不能作为opencode的插件来运行，那就只能将它的机制转换成hook来处理了。
以后可能还会有其他大模型的特殊机制。
但总的来说只要大模型的本质还是无状态概率相关性续写系统，那这个想法都是有现实意义的。

###opencode 上下文分层流水线改造 — 讨论纪要
####一、问题
opencode 的上下文管理是全局硬编码的统一管线——所有 LLM 共用同一套压缩/裁剪/序列化策略，无法针对特定模型做差异化优化。
####二、现实意义
只要 LLM 的本质还是无状态概率续写 + 按 token 计费，上下文组织方式就是"效果 × 成本"的核心杠杆。不同模型差异巨大：
模型	cache 特性
DeepSeek	prefix-cache，命中按 10% 计费
Claude	prompt caching，需 cache breakpoint 标记
Gemini	超长上下文（1M+）
OpenAI	无缓存折扣，窗口有限
现状： 所有模型走"最小公分母"策略，对哪个都不是最优的。
####三、灵感来源：Reasonix 的 3-Region 架构
Reasonix 围绕 DeepSeek 的 prefix-cache 机制设计了三级上下文分区：
┌─────────────────────────────────────┐
│  IMMUTABLE PREFIX（不可变前缀层）     │ ← 整个会话不变
│  system + tool_specs + few_shots    │   缓存命中靶子
├─────────────────────────────────────┤
│  APPEND-ONLY LOG（只追加日志层）      │ ← 只能追加，绝不重排/压缩
│  [user₁][assistant₁][tool₁]...      │
├─────────────────────────────────────┤
│  VOLATILE SCRATCH（易失草稿层）       │ ← 每轮重置，永不上传
│  R1 思考链、临时 plan state          │
└─────────────────────────────────────┘
效果：长会话缓存命中率 85%+，极端场景 *99.82%*。
####四、解决方案：分层流水线 Pipeline + Hook 架构
把现在的黑盒管线拆成透明的中间件栈，每层可被 plugin hook 拦截和修改：
Layer 1: 原始消息收集
  ├── 用户输入 + tool results + 系统事件
  └── 🔌 hook: onMessages(ctx) → mutate
Layer 2: 组合与编排
  ├── system prompt + instructions + skills + AGENTS.md
  └── 🔌 hook: onAssemble(ctx, modelId) → mutate
Layer 3: 历史窗口管理（核心差异层）
  ├── 压缩/裁剪/摘要（按模型策略不同）
  └── 🔌 hook: onWindow(ctx, modelId) → mutate
      ├── DeepSeek → 跳过压缩，append-only
      ├── Anthropic → 注入 cache breakpoint
      └── 默认 → 现有压缩策略
Layer 4: 格式转换
  ├── 序列化成各 provider 的 API schema
  └── 🔌 hook: onSerialize(ctx, provider) → mutate
Layer 5: 发送前最终调整
  ├── 注入缓存标记、调整 token 预算
  └── 🔌 hook: onBeforeSend(ctx, modelId) → mutate
                        ▼
                     发送
核心价值： 按 modelId 分流，Layer 3 不同模型走不同策略，互不干扰。默认行为 = 现在的硬编码管线，空 hook = 不变。向后兼容。
####五、实现方式
Git 分支策略：
main（fork 的默认分支）
  └── 纯粹跟踪 upstream/dev，不改一行代码
context-pipeline-layers（功能分支）
  └── 所有改动都在这里，定期 rebase main
流程：git pull upstream dev → git rebase main → 解冲突 → git push --force-with-lease
可干预程度评估：
层级	当前 plugin hook	能做什么
配置	compaction 选项	关闭自动压缩
指令	skill	引导模型行为
plugin	messages.transform	发送前改写消息
plugin	session.compacting	注入自定义上下文
引擎	—	需要 fork 改源码
####六、分支名建议
- context-pipeline-layers
- layered-context-pipeline
- context-middleware-hooks
####七、尚未解决的问题
- volatile scratch 区隔离需要引擎层支持，plugin 层面做不到
- hook 里能否拿到当前请求的 modelId 信息（需要确认 engine 是否暴露这个数据）
- 三层标记（immutable / append-only / scratch）需要从消息结构体层面设计
