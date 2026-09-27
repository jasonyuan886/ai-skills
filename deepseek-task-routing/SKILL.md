---
name: deepseek-task-routing
description: |
  什么任务该分流给DeepSeek API而不是用Claude额度，怎么调用。
  适用场景：结构化数据处理/分类打分、代码生成调试、批量文本摘要/提取、中文内容处理等
  对话理解要求不高但量大的任务，用便宜的DeepSeek顶上，省Claude额度做真正需要Agent能力的事。
  状态：2026-09-27袁总拍板的任务分流策略，API连通性已验证，具体任务效果仍在各组积累中——
  用完请把真实效果记进各自EVOLUTION_LOG.md，好的用法会被回填进这份技能。
---

# DeepSeek 任务分流策略

## 核心原则
不是所有任务都需要Claude的Agent能力（工具调用/多步骤推理/理解复杂业务铁律）。
凡是"规则明确、不需要判断该不该做、纯粹执行"的批量/结构化任务，优先分流给DeepSeek，省Claude额度。

## 已验证：API连通性
- `DEEPSEEK_API_KEY`（`/root/credentials_env.sh`）
- Endpoint: `https://api.deepseek.com/chat/completions`，OpenAI兼容格式
- Model: `deepseek-chat`（2026-09-27实测返回200，连通正常）
- 调用示例：
```bash
source /root/credentials_env.sh
curl -s https://api.deepseek.com/chat/completions \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-chat","messages":[{"role":"user","content":"你的prompt"}]}'
```
- 用其他模型名（如deepseek-reasoner/deepseek-coder）之前，先查DeepSeek官方文档确认当前实际可用的模型名，不要照抄没验证过的名字
- 另有`New-API`网关账号(admin/见credentials_env.sh)理论上可以聚合多个LLM provider，但2026-09-27实测本机3000端口未监听，**这个网关当前是否在跑、跑在哪，还没验证清楚，不要假设它能用**，直接用上面的DeepSeek官方API更可靠

## 适合分流给DeepSeek的任务类型
- 结构化数据分类/打分（比如按预设规则给候选项打分、贴标签）
- 代码生成/调试类子任务
- 批量文本摘要、信息提取（比如从网页/邮件里抽取结构化字段）
- 中文内容处理
- 数学/逻辑推理类子任务

## 不适合分流的（继续用Claude）
- 需要多步骤工具调用的Agentic任务（浏览器自动化、跨系统操作、SSH/脚本执行）——这是Claude Code的核心能力，DeepSeek没有对应的工具调用生态接入这套系统
- 面向海外客户的英文文案（冷邮件个性化、客户回复）——已有铁律要求"逐个真人手写体现真实了解"，需要高质量、上下文敏感的判断，不分流
- 任何需要理解复杂业务铁律、判断"能不能做/该不该做"的场景（比如要不要发这封邮件、这个内容合不合规）

## 分流前先确认两件事（2026-09-27，freshlock+xuanpin评估后回填的真实教训）
1. **目标逻辑本来是不是真的在吃Claude额度**——很多"看起来像打分/分类"的环节，本来就是纯脚本规则代码（正则/字符串匹配/算术），从来没调用过Claude，谈不上"分流"，硬接DeepSeek反而是新增一笔调用成本，不是省钱。先查清楚现状，不要看任务名字就假设它在吃额度。
2. **DeepSeek端有没有对应的真实数据能力**——DeepSeek的chat completions接口不联网搜索，如果任务需要真实外部数据（比如"这个产品社媒声量怎么样"这类需要实时信息的判断），DeepSeek只能凭训练数据编一个听起来合理的答案，不是真实数据。如果项目本身有"抓不到真实数据就如实标注不足、不装样子硬打分"这类规矩，这种场景不适合分流，宁可维持现状也不能用看似省钱的假动作牺牲数据可信度。

不满足以上两点的场景，不用为了"响应号召"硬分流。

## 使用后请回填
用了DeepSeek之后（或者评估完决定不用），把真实结论（省了多少额度/质量够不够/踩了什么坑，或者"为什么这次不适合分流"）记进自己项目的 `EVOLUTION_LOG.md`。这份技能目前只验证了API连通性，具体"在什么任务上效果好"还在各组积累实战数据，好的用法（以及不适用的边界）会被持续回填进这份技能（本库入库标准是只收实战验证过的经验，见本仓库README「入库标准」章节）。
