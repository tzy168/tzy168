# Agent 落地坑清单

> 目标：不仅知道概念怎么用，还要知道上线后会踩什么坑。  
> 用法：做项目前扫一遍对应章节，做完后回来勾选是否遇到过。

---

## 一、Prompt 与 LLM 输出

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| Prompt 脆弱 | 改一句系统提示，行为大变 | LLM 对措辞极度敏感 | 固定评测集，改 prompt 必跑回归 |
| 输出格式不稳定 | 要求 JSON，却返回 Markdown 代码块 | 模型不保证 structured output | 用 JSON mode / response_format；解析失败 retry + 降级 |
| 幻觉 | 编造法规、接口、数据 | LLM 本质是概率生成 | RAG 引用溯源；关键事实走 Tool 查库，不让模型「猜」 |
| 上下文遗忘 | 多轮对话后忘记早期约束 | Context Window 有限 | 滑动窗口 + 摘要；System Prompt 放核心约束 |
| Token 爆炸 | 账单突然很高 / 请求超时 | 历史消息、Tool 结果全塞进去 | 截断策略；Tool 返回只留摘要；设 max_tokens |
| 中英文混用失控 | 用户中文问，模型英文答 | 系统提示或训练偏好 | System Prompt 明确语言；Few-shot 给中文示例 |
| 温度参数乱设 | 同一问题每次答案差很大 | temperature 过高 | 工具调用 / RAG 用低温度（0~0.3）；创意场景才调高 |
| 模型版本漂移 | 线上行为突然变了 | 厂商 silent update 或换模型 | 锁定模型版本号；变更走灰度 + 评测 |

---

## 二、Function Calling / Tool Loop

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 无限循环 | Agent 反复调同一个 Tool | 缺少终止条件 | 设 `max_iterations`（通常 5~15）；检测重复调用 |
| Tool 选错 | 该搜数据库却去调 HTTP | Tool 描述不清晰 | 每个 Tool 写清「何时用 / 何时不用」；负面示例 |
| Tool 参数幻觉 | 编造不存在的 file_path、id | 模型填参 | Pydantic / Zod 校验；非法参数直接报错让模型重试 |
| 解析失败 | Tool Call JSON 残缺 | 流式输出截断、模型格式错误 | 非流式拿 tool_calls；解析失败 retry 1~2 次 |
| 串行太慢 | 5 个 Tool 串行跑 30 秒 | 默认 one-by-one | 无依赖的 Tool 并行；设单 Tool 超时 |
| Tool 超时无兜底 | 整个 Agent 卡死 | 外部 API 慢或挂 | 单 Tool timeout + 超时返回结构化错误给 LLM |
| Tool 返回过大 | 撑爆 context | 直接塞原始 HTML / 大 JSON | 截断 + 摘要；只返回 LLM 需要的字段 |
| 副作用重复执行 | 同一「发邮件」Tool 调了 3 次 | 模型 retry 或 loop | 幂等设计；写操作加 confirm / Human-in-the-loop |
| 权限越界 | Agent 删了不该删的文件 | Tool 权限过大 | 最小权限；敏感操作白名单 + 人工审批 |
| 错误信息回传不当 | LLM 看到 stack trace 后更乱 | 原始异常直接塞给模型 | 转成结构化 `{ error, suggestion }` |
| 国内外 Tool 定义差异 | 同一套 schema 在 DeepSeek / Qwen 表现不同 | 各厂商 FC 实现细节不同 | 每个目标模型单独测 Tool Calling；留 adapter 层 |

---

## 三、RAG（检索增强生成）

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 切片太碎 | 召回的 chunk 没有完整语义 | chunk_size 过小 | 按段落 / 标题切；300~800 token 起步，按文档类型调 |
| 切片太大 | 召回噪声多、浪费 token | chunk 含多个主题 | 语义切分；重叠窗口（overlap 10~20%） |
| 召回不到 | 明明库里有，就是搜不到 | Embedding 模型与 query 语言不匹配；切片策略差 | 换中英多语 Embedding；混合检索（向量 + 关键词 BM25） |
| 召回太多噪声 | 答非所问 | Top-K 过大；无 rerank | K 先取 10~20，rerank 后留 3~5 |
| 无引用溯源 | 用户不信、无法 debug | 只把 chunk 塞 prompt 不展示来源 | 回答附 `[1][2]` 引用；前端展示原文片段 |
| Embedding 与 LLM 语言不一致 | 中文文档 + 英文 Embedding 模型 | 模型选型随意 | 中文场景用 bge / text-embedding-v3 等多语模型 |
| 文档更新后答案旧 | 用户看到过期信息 | 向量库未同步 | 增量索引 + 版本号；回答注明「数据截至 xxx」 |
| PDF 解析烂 | 表格乱、页眉页脚干扰 | 直接用纯文本提取 | 专用解析（Unstructured / MinerU）；表格单独处理 |
| 相似度阈值乱设 | 要么全拒答要么全胡答 | 固定 threshold 不适配所有 query | 低置信度时明确说「未找到」而非硬答 |
| 多文档冲突 | 两份文档说法矛盾 | 无冲突检测 | 检索结果标注来源；冲突时让 LLM 说明或追问用户 |
| 索引成本高 | 重建索引几小时 | 全量重跑 | 增量更新；按 collection 分片 |
| pgvector 维度不对 | 插入报错 | Embedding 维度和表定义不一致 | 建表时锁定模型维度（如 1024 / 1536） |

---

## 四、Agent 编排（LangGraph / 多 Agent）

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 状态膨胀 | 内存占用高、序列化失败 | State 塞了全量历史 | 只存必要字段；历史存 DB，State 存指针 |
| 图过于复杂 | 改一处崩全局 | 节点耦合、无边界 | 单职责节点；子图拆分 |
| 死锁 / 无出口 | 永远停在某个节点 | 条件边写错 | 每个分支都要有 END 或 max_step |
| 多 Agent 扯皮 | 互相推活、循环对话 | 角色边界模糊 | 明确每个 Agent 的输入输出契约 |
| Human-in-the-loop 卡住 | 等审批等到超时 | 无超时 / 无默认动作 | 审批超时策略（跳过 / 降级 / 通知） |
| 并发写 State | 数据竞争 | 多 Agent 同时改状态 | 单写者模式；或用 reducer 合并 |
| Checkpoint 丢失 | 重启后对话断档 | 未持久化 checkpoint | LangGraph 接 Postgres / Redis 存 checkpoint |
| 调试困难 | 不知道哪步出错 | 无 trace | 每节点打 log；接 Langfuse 看完整链路 |

---

## 五、记忆（Memory）

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 短期记忆撑爆 | 长对话后请求失败 | 全量 messages 发送 | 摘要压缩；只保留最近 N 轮 + 摘要 |
| 长期记忆乱召回 | 召回了无关的旧对话 | 向量检索阈值过低 | 按 user_id / session 隔离；相似度过滤 |
| 记忆污染 | 错误信息被记住并反复引用 | 无校验就写入 | 写入前过滤；支持用户「忘记」 |
| 会话混淆 | A 用户看到 B 的上下文 | session 隔离 bug | 严格 session_id / user_id 隔离；代码 review 重点 |
| 摘要丢细节 | 用户说「按刚才那个方案」模型不懂 | 过度摘要 | 关键实体（人名、数字、ID）单独存 structured memory |

---

## 六、流式输出与前端（前端转 Agent 高发区）

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| SSE 断连 | 流中途卡住 | 代理超时、Nginx buffer | 设 `X-Accel-Buffering: no`；心跳 ping |
| 流式 + Tool Call 冲突 | 前端解析乱 | Tool 在流式 chunk 里不完整 | Tool 决策阶段非流式；仅最终回答流式 |
| Markdown 渲染闪屏 | 代码块 / 公式跳来跳去 | 流式逐字渲染 | 节流渲染（requestAnimationFrame）；或用 stable markdown 库 |
| 重复渲染 | 同一消息渲染多次 | React key 或 state 更新逻辑 | 消息 id 去重；delta 追加而非全量替换 |
| 取消无效 | 用户点停止仍在跑 | 只断前端，后端继续 | AbortController 传到底层 LLM 请求 |
| 并发请求 | 用户连点发送，答案交叉 | 无 in-flight 锁 | 发送中 disable；或支持多 session |
| Tool 可视化滞后 | 用户看不到 Agent 在干嘛 | 只展示最终文本 | 中间步骤（thinking / tool_call / tool_result）单独 UI 层 |
| 长回答卡顿 | 滚动掉帧 | 一次 insert 上万字 DOM | 虚拟列表；分段 append |
| CORS / Cookie | 本地好使，部署跨域失败 | 前后端分离配置 | 明确 credentials 策略；生产用同域或 BFF |

---

## 七、API 与模型接入（国内环境）

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 速率限制 429 | 高峰批量失败 | 免费 / 低配 quota | 指数退避 retry；队列削峰；多 key 轮换（合规前提下） |
| 国内模型 FC 格式差异 | 换模型后 Tool 全挂 | OpenAI 兼容但不完全兼容 | 抽象 LLM 层；每个模型单独集成测试 |
| 网络不稳定 | 偶发 timeout | 跨境 / 运营商 | 国内模型优先；timeout + retry；熔断 |
| API Key 泄露 | 账单暴增 | Key 写前端 / 提交 Git | Key 仅后端；`.env` + gitignore；CI 扫描 |
| 成本失控 | 月账单超预期 | 无用量监控 | 按 user / 接口统计 token；设日预算告警 |
| 模型下架 | 线上突然 404 | 依赖单一模型 | 抽象层支持 fallback 模型 |
| 内容审核拦截 | 正常业务被拒 | 国内合规过滤 | 输入输出过审；敏感词预处理；友好错误提示 |
| 发票 / 企业认证 | 个人 key 无法报销 | 国内云企服流程 | 早期个人开发，上线切企业账号 |

---

## 八、数据库与存储

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 会话表暴涨 | DB 磁盘满 | 每条 message 全量存、无清理 | TTL 归档；冷热分离 |
| JSON 字段查询慢 | 日志检索卡 | 全塞 JSONB 无索引 | 常用字段拆列；GIN 索引 |
| 向量检索慢 | RAG 延迟 >3s | 无索引 / 数据量大 | IVFFlat / HNSW 索引；数据量大了换专用向量库 |
| 事务与 LLM 不一致 | Tool 写库成功但 LLM 以为失败 | 无 idempotency | 业务 id + 幂等键；先 DB 后通知 LLM |
| 迁移踩坑 | Embedding 模型换了旧向量废了 | 向量不可跨模型复用 | 模型版本字段；换模型触发 re-embed |
| 连接池耗尽 | 并发高时 500 | FastAPI 同步阻塞 / 池太小 | 异步驱动；合理 pool_size |

---

## 九、安全与合规

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| Prompt 注入 | 用户输入覆盖系统指令 | 「忽略以上规则…」 | 输入 sanitization；System 与 User 严格分隔 |
| 间接注入 | 网页 / 文档里藏恶意指令 | RAG 召回了攻击文本 | 检索结果当 untrusted input；不直接执行 |
| SSRF | Agent 读内网 URL | fetch Tool 无限制 | URL 白名单；禁止私有 IP 段 |
| 代码执行 | Agent 跑恶意代码 | 给了 shell / python exec | 沙箱（Docker / WASM）；禁网或限网 |
| 数据出境 | 敏感数据发到境外 API | 用了 OpenAI 传国内用户数据 | 国内数据走国内模型；脱敏后再调用 |
| 日志泄密 | 日志里打了完整 prompt | debug 级别过高 | 生产脱敏；PII 过滤 |
| 越权访问 | 用户 A 查用户 B 的知识库 | 鉴权漏了 | 每个 RAG 查询带 tenant_id 过滤 |

---

## 十、部署与运维

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| Docker 镜像巨大 | 构建 / 拉取慢 | 塞了 torch 全量 | 多阶段构建；CPU 版依赖；.dockerignore |
| 冷启动慢 | 首请求 30s+ | 模型 / 依赖懒加载 | 健康检查预热；keep-alive |
| 环境变量遗漏 | 生产启动失败 | `.env` 本地有、服务器没配 | 启动时校验 required env；用配置模板 |
| 无健康检查 | K8s 把还在启动的 pod 接流量 | 缺 liveness / readiness | `/health` 端点；依赖就绪再接收请求 |
| 日志不可查 | 出 bug 无法复现 | 无 request_id | 全链路 trace_id；Langfuse / 结构化日志 |
| 无降级 | LLM 挂了全站 500 | 单点依赖 | 缓存常见问答；LLM 不可用时友好降级文案 |
| CI 没测 Agent | 改 prompt 上线后翻车 | 无自动化评测 | 固定 golden set；CI 跑 pass rate 阈值 |
| 版本不可回滚 | 出问题只能硬改 | 无镜像 / 配置版本 | 每次发布打 tag；DB migration 可逆 |

---

## 十一、评测与迭代

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 凭感觉改 prompt | 改完不知道好坏 | 无评测集 | 至少 20 条真实 query + 期望行为 |
| 只看最终答案 | Tool 选错但没发现 | 只评 text 不评 trace | 评 tool_selection、latency、token 用量 |
| 过拟合 demo | demo 完美，真实用户崩 | 评测集太理想 | 加入边界 case、脏输入、 adversarial |
| 无 A/B | 不知道哪个版本更好 | 缺对比机制 | Langfuse experiment 或简单 split |
| 忽略延迟 | 答案对但用户流失 | 只优化准确率 | 设 P95 延迟预算；复杂任务异步通知 |

---

## 十二、低代码平台（Dify / Coze）特有问题

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 平台锁死 | 想迁出自建很困难 | 深度绑定平台数据格式 | 早期就导出 workflow 定义；核心逻辑留代码层 |
| 调试黑盒 | 节点失败不知道哪步 | 可视化但 trace 弱 | 关键链路用代码实现；平台只做编排 |
| 版本不可控 | 平台升级行为变了 | SaaS 更新 | 自部署 Dify；锁定版本 |
| 性能天花板 | 并发一高就慢 | 共享资源 | 核心路径自建；平台做原型 |

---

## 十三、认知与团队坑（转岗常见）

| 坑 | 表现 | 原因 | 应对 |
|----|------|------|------|
| 简历堆概念 | 面试深挖就穿 | 只看不练 | 每个概念对应一个可 demo 的小模块 |
| 忽视评测 | 「能跑」当「能用」 | Demo 思维 | 上线标准：准确率 + 延迟 + 成本三角 |
| 重新造轮子 | 手写 Agent Loop | 不了解 LangGraph 等 | 先框架后自研；自研只解决框架解决不了的 |
| 前端优势浪费 | 只做后端 Agent | 不知道 UI 也是壁垒 | 流式 + Tool 可视化是你的差异化 |
| 只学 OpenAI | 国内面试 / 部署脱节 | 教程偏国外 | DeepSeek / Qwen 为主力；OpenAI 作了解 |
| 项目太散 | GitHub 一堆半成品 | 没有主线 | 1 主 1 副，做到可部署可讲架构 |

---

## 快速自检（上线前 5 分钟）

```
□ max_iterations / timeout 设了吗？
□ Tool 参数有 schema 校验吗？
□ API Key 只在服务端吗？
□ RAG 低置信度时会拒答吗？
□ 有引用溯源吗？
□ 流式断连 / 取消处理了吗？
□ 会话按 user 隔离了吗？
□ 有 token 用量统计吗？
□ 有至少 10 条 golden 测试吗？
□ 敏感 Tool 有 Human-in-the-loop 吗？
```

---

## 相关文件

- [Agent 核心概念](./README.md)
- [学习路线](./roadmap.md)（待建）
- [项目规划](./projects.md)（待建）
- [国内落地](./china.md)（待建）
