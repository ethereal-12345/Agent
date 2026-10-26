Agent任务规划
一、核心概念
任务规划（Task Planning）是Agent的核心能力：把复杂、模糊、大目标，拆成一系列可执行、有序的子任务，再依次调用工具/LLM完成。
 
本质：LLM当“调度大脑”，先想步骤，再干活；区别于直接一次提问、Function Calling只做单次工具调用。
 
- 输入：用户复杂目标
- 输出：结构化子任务列表、依赖关系、执行顺序
- 定位：属于Agent的思考层，在工具调用之前
 
二、主流规划范式
 
1. 链式规划（Sequential / 线性规划）
 
子任务按顺序执行，后面任务依赖前面结果。
示例：用户问“分析这份财报PDF，画营收趋势，写总结”
 
1. 解析PDF，提取营收数据
2. 清洗数据，提取时间+收入
3. 生成图表
4. 撰写分析报告
✅ 简单稳定，适合流程固定场景
❌ 不支持分支、回退、并行
 
2. 树状规划（Tree Planning）
 
主任务拆成多个并行分支，分支再继续拆解，形成任务树。
适合多子目标独立：比如同时检索文档、查数据库、搜参考资料。
可以部分子任务并行执行。
 
3. 递归规划（Recursive Planning）
 
子任务如果仍然太难，继续调用规划器二次拆解，直到任务足够小、可以直接调用工具。
 
一句话：拆到“原子任务”为止。
原子任务：单一动作，一般对应一次Function Calling（检索、查询、计算、读取文件）
 
4. 反馈式规划 / 重规划（Re-planning）
 
执行中发现失败、结果不对、用户需求变化 → 暂停，重新修改任务清单。
Agent最关键的工程能力，避免一条路走到黑。
触发重规划条件：
 
- 工具返回报错、无结果
- 返回结果和目标明显不符
- 出现新信息，改变原有目标
- 子任务执行超时
 
三、经典规划算法
 
1. Plan-and-Solve（PaS）
先生成完整计划，再一步步执行；适合目标清晰的任务。
Prompt模板：先列出所有步骤，再执行。
优点：步骤全局可见，便于调试；缺点：一开始计划错了，后面全部跑偏。
2. Reasoning and Acting（ReAct）
思考→行动→观察循环，边做边规划，不是一次性输出全部计划。
 Thought → Action → Observation 
适合环境不确定、需要根据返回信息调整的场景（最常用）。
3. Tree of Thoughts（ToT，思维树）
一个任务生成多个候选分支方案，对每个分支评估打分，择优继续探索，类似搜索。
适合数学、推理、最优方案选择；算力开销大。
4. Graph of Thoughts（GoT）
任务之间可以有依赖、汇聚、并行，用图结构表达，比树更灵活。
 
四、任务拆解的标准原则
 
1. 原子性：一个子任务只做一件事，尽量对应单个工具调用
2. 可验证：每个子任务有明确成功/失败判定标准
3. 依赖清晰：标注哪些任务要等前面结果
4. 最小粒度，不要过度拆分：拆太碎会增加token、多轮调用成本
5. 保留目标锚点：每一步都对照原始用户需求，防止偏离
6. 容错预留：每个步骤可以设置重试、重规划分支
 
❌ 反面例子：一个任务“处理文档并回答所有问题”（太大，不是原子任务）
✅ 正面例子：“读取文档 → 检索相关片段 → 计算指标 → 整理答案”
 
五、Prompt设计要点
 
固定结构一般包含：
 
1. 总目标
2. 可用工具列表
3. 输出格式（JSON优先，方便程序解析）
4. 约束：遇到失败如何处理、是否允许重规划
5. 输出字段示例： task_id、task_name、description、deps、status 
 
示例输出结构：
 
json
  
{
  "plan": [
    {"id":1,"name":"解析PDF","dep":[]},
    {"id":2,"name":"检索营收数据","dep":[1]},
    {"id":3,"name":"生成图表","dep":[2]}
  ]
}
 
 
优先结构化JSON输出，不要纯自然语言列表，程序很难解析。
 
六、和Function Calling的关系
 
- 任务规划：决定做什么、顺序是什么（思考层）
- Function Calling：执行单个原子任务（动作层）
- 关系：规划产出任务清单，再循环调用工具执行；执行观测结果回传给规划器，支持重规划。
 
七、常见坑
 
1. 一次性规划过长，LLM脑补不存在的步骤（计划幻觉）
2. 依赖关系写错，并行任务提前执行
3. 不会重规划：失败后继续按旧计划硬跑
4. 拆分太细，轮次爆炸、token成本飙升
5. 规划器分不清“需要工具”和“LLM自己就能推理”的任务
 
八、适用场景
 
✅ 复杂知识库问答、数据分析Agent、自动化文档处理、多工具串联工作流、代码执行Agent
❌ 简单一问一答（不需要规划，直接调用LLM即可）
 
规划 = 把大目标拆成有序原子任务；
Plan-and-Solve：先全计划再跑；ReAct：边跑边思考；ToT：多分支搜索择优；遇到失败触发重规划。




 
工具调用 Function Calling / Tool Use 
一、基础概念
Function Calling（工具调用）：大模型不直接生成答案，而是输出结构化参数，交给外部函数执行，拿到返回结果后再继续推理回答。
 
核心：LLM负责判断要不要调用、选哪个工具、填参数；真实逻辑、IO、计算、查询都在外部代码执行。
 
- Tool Use：广义叫法；Function Calling 是 OpenAI 最先推出的标准化实现，现在大家基本混用
- 和普通Prompt区别：普通LLM凭参数内知识回答；工具调用可以对接外部真实世界数据、能力
 
二、完整执行流程（标准循环）
 
1. 用户提问 + 系统提示词（附带工具描述schema）
2. LLM判断：是否需要调用工具？选哪个tool？生成入参
3. 后端解析模型输出的结构化JSON，校验参数
4. 本地执行函数（检索、查库、计算、调用接口）
5. 把工具返回结果（Observation）塞回上下文
6. LLM结合工具结果，整理自然语言回复，或继续调用下一个工具（多轮连续调用）
 
循环：User → LLM(Thought/ToolCall) → 代码执行 → Observation → LLM
 
三、工具Schema
用JSON Schema描述工具，告诉模型：工具名字、功能说明、入参名称、类型、是否必填、描述。
示例结构：
 
json
  
{
  "type": "function",
  "function": {
    "name": "search_knowledge",
    "description": "检索知识库文档，用于回答资料相关问题",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {"type":"string","description":"检索关键词"}
      },
      "required": ["query"]
    }
  }
}
 
 
✅ 写Schema要点：描述清晰、用词简洁；参数说明要让模型知道该填什么；不要写歧义描述
 
四、两种调用模式
 
1. 单轮调用：一次问题只调用一次工具，拿到结果直接回答（简单场景：查天气、单条RAG检索）
2. 链式/连续多轮调用：调用A工具拿到结果，再用结果作为参数调用B工具（Agent任务拆解后的流水线）
 
例：先查订单 → 再查订单关联物流
 
五、关键区分：Function Calling / ReAct / 任务规划
 
- Function Calling：原子动作单元，单次工具调用能力
- ReAct：Thought → Action(tool call) → Observation 的循环范式，依赖工具调用
- 任务规划：提前生成一整套任务清单，规划里面每个原子任务大多落地为一次Function Calling
 
一句话：规划决定调用顺序，Function Calling是单次动作
 
六、能力边界 & 模型侧差异
 
- OpenAI、DeepSeek、Qwen、GLM都支持Function Calling；底层实现略有差别，schema基本兼容
- 模型不会执行代码！模型只输出参数，执行永远在后端（很多新手误区）
- 模型可能：选错工具、参数填错、编造参数（工具幻觉）
 
七、工程校验层
模型输出不可信，后端必须加校验：
 
1. JSON解析捕获异常（输出格式乱了）
2. 参数类型校验、范围校验、必填项校验
3. 权限校验（防止越权调用高危工具：删数据、执行shell）
4. 超时控制、重试策略
5. 工具返回结果清洗，过长结果截断，避免上下文爆掉
 
八、常见工具类型
- 检索类：RAG知识库检索、向量库查询
- 查询类：MySQL/Redis查询、API接口请求
- 计算类：计算器、代码解释器（Python Sandbox）
- 读写类：文件读取、文档解析
- 多媒体：Whisper转写、TTS生成、VLM图片理解
 
九、常见坑
1. 工具幻觉：模型编造不存在工具、编造参数、编造返回内容，不真实调用
缓解：schema精简、强格式约束、结构化输出、后端校验
2. 工具描述写太长：模型理解混乱，选错工具
3. 工具返回内容过大：token暴涨、成本高、容易丢失信息 → 摘要、截断、重排
4. 循环死调用：工具返回结果已经足够回答，模型还反复调用同一个工具
缓解：系统提示词增加终止条件、设置最大连续调用次数上限
5. 入参语义理解偏差：参数描述模糊，模型传错query
 
十、最佳实践
 
1. 工具数量不要堆砌：一个Agent工具控制在10个以内，越多越容易选错
2. 高危工具增加二次确认（如删除、修改数据）
3. 工具返回结果尽量结构化（Markdown/JSON），方便LLM阅读
4. 日志记录：用户query、tool_name、入参、返回值，方便调试
5. 区分：哪些问题不需要工具，LLM原生知识就能回答，避免不必要调用，节省成本
 
十一、极简伪代码流程
 
plaintext
  
tools = [schema1, schema2]
resp = llm.chat(user_query, tools=tools)
if resp.is_tool_call:
    name = resp.tool_name
    args = resp.tool_args
    result = dispatch(name, args) # 分发到对应函数执行
    llm.chat(..., tool_result=result)
else:
    return resp.content
 



 Agent 记忆管理（短时记忆 + 长时记忆）
一、核心概念
记忆是Agent用来保存历史信息、上下文、用户偏好、过往任务结果的模块，解决“Agent聊完就忘”的问题。
分为 短时记忆（工作记忆）、长时记忆（持久记忆），还有中间层的语义记忆/向量记忆。
 
类比人脑：短时=当下正在思考的内容；长时=很久之前记住的知识、偏好、历史记录。
 
二、短时记忆（Working Memory / 上下文记忆）
 
1. 定义
 
保存在当前对话上下文窗口内，随轮次一起传给LLM；生命周期=单次会话，会话结束即清空（除非落地保存）。
 
- 本质：对话历史消息列表  [ {"role":"user"}, {"role":"assistant"}, {"tool_result":...} ] 
- 承载内容：用户问题、Agent思考、工具调用、工具返回observation、中间计划
 
2. 局限
 
- 受上下文窗口上限约束，轮次一多就溢出
- 超过窗口后，早期内容直接丢失，LLM遗忘前面信息
- token成本随对话变长持续上涨
 
3. 短时记忆优化手段
 
1. 滑动窗口：只保留最近N轮，旧对话直接丢弃（简单粗暴，会丢失早期信息）
2. 摘要压缩（上下文压缩）：把久远的多轮对话浓缩成简短摘要，替换原始历史
3. 重要性过滤：丢弃无意义闲聊、重复工具返回，保留关键结论
4. 结构化沉淀：把关键结论提取成键值对，代替大段原文
 
短时记忆是Agent思考的“草稿纸”，所有ReAct、任务规划、工具调用循环都依赖它。
 
三、长时记忆（Long-term Memory）
 
1. 定义
 
独立于LLM上下文窗口，持久化存储在外部数据库；跨会话、跨时间复用，需要的时候才检索加载进短时上下文。
存储介质：关系库、Redis、向量库、KV数据库。
 
- 什么时候触发：用户新开对话、隔很久再来，依然可以召回之前的信息
 
2. 长时记忆三大分类
 
1. 陈述式记忆（语义记忆，向量记忆）
内容：用户历史提问、历史结论、文档知识、过往任务记录
实现：文本Embedding → 存入向量库；用户提问时，语义检索召回相关记忆，追加到上下文
典型：RAG本质就是一类语义长时记忆
2. 程序记忆（习惯/技能）
内容：固定流程、用户偏好设置（比如“用户喜欢简洁回答、不要表格”）
实现：结构化KV存储（MySQL/Redis），直接匹配读取，不需要向量检索
3. 事件记忆（情景记忆）
内容：历史事件、某次对话发生了什么、任务执行记录
实现：结构化日志 + 向量双存；既能按时间筛选，也能语义检索
 
3. 完整读写流程
 
- 写入（记忆沉淀）：一轮对话结束 → 判断信息是否值得存 → 格式化/摘要 → 入库（向量库+结构化库）
- 读取（记忆召回）：用户新query → 检索长时记忆 → 筛选相关记忆 → 注入短时上下文 → LLM思考
 
重点：长时记忆不会全部塞进上下文，只召回相关片段，避免窗口爆炸。
 
四、短时记忆 vs 长时记忆对比
 
           短时记忆                     长时记忆 
存放位置   LLM消息列表（prompt内）     外部数据库（向量库/MySQL） 
生命周期  单次会话，会话结束消失         持久保存，跨会话 
读取方式    全部直接给模型               检索后选择性加载 
容量      很小，受上下文窗口限制          几乎无上限 
成本       高，占用token                检索成本低，不占用上下文直到召回 
用途  当前思考、工具循环、本次任务规划    用户偏好、历史对话、知识库、过往任务 
 
五、记忆管理核心子能力
1. 记忆提取（Memorize）
不是所有对话都要存！增加判断：是否是有效信息、用户偏好、重要结论；过滤闲聊、临时工具中间结果。
Prompt交给LLM识别：哪些内容值得保存为长期记忆。
2. 记忆检索（Retrieve）
混合检索：关键词过滤 + 语义向量检索 + 时间过滤；控制召回数量Top-K。
3. 记忆评分&过滤
召回多条记忆后，用Reranker或者LLM打分，剔除无关、过时、冲突记忆。
4. 记忆更新/遗忘（Forget）
非常关键：信息会过时、会矛盾。
策略：
 
- 覆盖更新：用户修改偏好，旧记忆标记失效
- 衰减遗忘：长期没有访问的记忆降低权重，慢慢不再召回
- 主动删除：用户指令删除记忆
 
六、经典架构：记忆流水线
 
plaintext
  
用户输入 → 短时上下文（消息列表）→ Agent思考/工具调用
一轮结束 → 记忆提取器 → 摘要/结构化 → 写入长时库
新一轮用户输入 → 检索长时记忆 → 筛选、重排 → 注入短时上下文 → LLM
 
 
七、高频坑点
1. 无脑全量存储所有对话：记忆库爆炸、召回大量无关冗余信息
2. 召回太多记忆塞进上下文：token暴涨、信息干扰、幻觉变多
3. 新旧记忆冲突，没有冲突检测：Agent前后回答矛盾
4. 只做向量检索，没有结构化KV：偏好类信息检索不稳定
5. 遗忘机制缺失：过时信息一直被召回，答案陈旧
6. 混淆记忆和RAG：RAG是外部知识记忆；Agent记忆偏向用户交互历史、用户偏好
 
八、适用场景
 
✅ 个人助手、长期客户对话Agent、多轮任务机器人、需要记住用户习惯的应用
❌ 一次性问答、单次任务、不需要跨会话的简单调用（不需要长时记忆）
 
九、
短时记忆 = 当前上下文草稿，供本轮思考；
长时记忆 = 外部持久仓库，按需检索召回；
记忆管理三件事：识别存什么、检索相关内容、过期/冲突记忆做遗忘更新。


ReAct工作流
一、核心定义 
ReAct = Reason + Act（推理 + 行动）
是Agent最经典的循环范式：让大模型交替进行思考推理和调用工具行动，再观察工具返回结果，循环直到完成目标。
 
区别于一次性先写完整计划再执行（Plan-and-Solve）：ReAct是边思考、边做、边看反馈，动态调整思路。
 
二、标准循环链路（Thought → Action → Observation）
1. Thought（思考）：LLM分析当前目标、已有信息，判断：我现在缺什么信息、要不要调用工具、调用哪个工具、参数是什么
2. Action（行动）：输出工具调用（Function Calling），交给后端执行
3. Observation（观察）：拿到工具返回结果、报错、空值等原始信息，放回上下文
4. 回到Thought，继续推理：判断信息够不够、是否继续调用工具 / 直接生成最终答案
 
循环终止条件：信息充足，不需要再调用工具，输出Final Answer
 
plaintext
  
User Query
→ Thought：我需要检索知识库
→ Action：调用search_knowledge(query="xxx")
→ Observation：检索到的文档片段
→ Thought：信息足够，可以回答
→ Final Answer
 
 
三、和其它范式对比
1. ReAct vs Plan-and-Solve（PaS）
 
- PaS：一次性先生成全部任务计划，再按顺序执行；适合目标稳定、确定的任务；缺点：一开始计划错了，全程跑偏，很难动态调整
- ReAct：不提前定死全部步骤，每一步基于上一步观察再决定下一步；适合开放、不确定、会遇到意外的场景（最常用）
 
2. ReAct vs CoT
 
- CoT：只有内部推理，没有外部工具行动，纯LLM脑补
- ReAct = CoT式思考 + 外部工具调用，推理可以落地到真实环境
 
3. ReAct vs ToT（思维树）
 
- ReAct：单条推理链，线性循环
- ToT：同时探索多条推理分支、打分择优，搜索开销更大
 
四、两种实现方式 
方式1：文本版ReAct
 
模型输出自然语言Thought，再输出Action，靠Prompt约束格式。
缺点：格式容易崩，解析麻烦，生产很少直接用。
示例输出：
 
plaintext
  
Thought: 用户问销量，我需要调用数据库查询工具
Action: db_query{"sql":"select sale from ..."}
Observation: 2300万
 
 
方式2：Function Calling 版ReAct（工业主流）
 
Thought放在模型内部隐式推理，直接输出标准tool_call，后端解析执行。
不再让模型手写Thought文本，更稳定，就是我们平时写的工具调用循环。
 
现在工程上说的ReAct，绝大多数都是基于Function Calling的循环。
 
五、Prompt核心约束要点
- 告诉模型：每一步只能二选一：①调用工具 ②输出最终答案
- 最大循环轮次限制（防死循环！比如最大5~10轮）
- 明确规则：Observation是客观结果，不能篡改、编造
- 禁止无意义重复调用同一个工具
 
六、优势
✅ 动态自适应：遇到报错、无结果、新信息，可以即时改变思路，天然支持重规划
✅ 符合人类解决问题习惯：想一下，动手试一下，看结果再想
✅ 落地成熟：大部分开源Agent框架（LangGraph、LlamaIndex）默认基于ReAct
✅ 降低幻觉：依赖工具返回的客观Observation，而不是模型内部知识
 
七、缺点 & 高频坑
 
1. 循环死锁：反复调用同一个工具、拿到相同结果还不停。
缓解：设置最大轮次、历史检测，如果连续两次action完全相同，强制终止
2. 短视：只看眼前一步，缺少全局规划，复杂多依赖任务容易绕路
缓解：ReAct + 前置任务规划混合方案（先粗规划，再ReAct逐步执行）
3. Thought幻觉：模型主观判断出错，选错工具、参数错误
4. 轮次多 → token成本上升、延迟变高
5. Observation过长：工具返回大量文本，上下文膨胀，需要摘要、截断、重排
 
八、混合进阶方案
Plan + ReAct：先做一次高层任务拆解（Plan），得到任务清单；再用ReAct循环逐个执行子任务，每一步可以根据结果动态调整/跳过子任务。
兼顾全局方向和局部灵活调整。
 
九、适用场景
✅ 知识库问答、数据查询Agent、代码执行Agent、多工具组合任务、不确定开放问题
❌ 流程高度固定、步骤完全确定的流水线（用固定工作流比ReAct更稳更便宜）
 
十、
ReAct = 思考→行动→观察循环；
CoT只动脑，ReAct动脑还能动手拿外部信息；
Plan-and-Solve先定全部路线再跑，ReAct走一步看一步。
 



LangChain 生态
说明：LangChain 核心定位：封装LLM、工具、检索、记忆，快速组装Agent与RAG流程；现在主流推荐 LangGraph 做Agent循环，LangChain 负责基础组件。
一、三大核心模块 
1. Chains 链
 
Chain = 把多个组件按顺序串联，定义固定工作流，输入→经过多步→输出。
 
- 本质：可调用单元  chain.run(input) 
- 基础链
-  LLMChain ：PromptTemplate + LLM，最基础，输入prompt给模型
-  SequentialChain ：串行链式，A输出作为B输入（适合固定顺序任务）
-  RouterChain ：路由链，根据用户问题分发到不同子链（问题分类分流）
- RAG高频链
-  RetrievalQA ：检索 + 组装上下文 + LLM回答（经典RAG链）
-  ConversationalRetrievalChain ：带对话记忆的RAG，会先基于历史改写query再检索
- 进阶： TransformChain  做数据清洗、格式转换； MapReduceChain  /  Refine  长文档摘要
 
局限：传统Chain大多是静态流程，分支、循环、回退不好写 → 这就是LangGraph解决的问题
 
2. Tools 工具
 
Tool 给Chain/Agent提供外部能力，对应Function Calling。
 
- 组成： name 、 description 、 args_schema （参数结构）、 func 执行函数
- 内置常用工具：
-  TavilySearch 联网搜索
-  PythonREPL 代码沙盒执行
-  RetrievalTool 知识库检索
-  Calculator 数学计算
- 自定义Tool： @tool 装饰器（最简单）、 StructuredTool 
 
python
  
from langchain.tools import tool
@tool
def add(a:int,b:int)->int:
    """两个数字相加"""
    return a+b
 
 
- Toolkit：一组相关工具打包（如SQLDatabaseToolkit，包含查表、查询、描述表）
- 注意：Agent负责选工具；Chain一般预先固定用哪个工具
 
3. Context Management 上下文管理
 
负责控制进入LLM窗口的内容，解决上下文溢出、历史管理、压缩，包含三块：
 
1. Chat Message History 对话历史
-  ChatMessageHistory ：底层消息存储
- 内存版 InMemoryChatMessageHistory （测试用，重启丢失）
- 持久化：Redis / Postgres / SQLite 存储消息
2. Memory 记忆封装（高层API）
-  ConversationBufferMemory ：直接存全部原始消息，最简单，容易爆窗口
-  ConversationSummaryMemory ：自动LLM摘要历史，节省token
-  ConversationBufferWindowMemory ：滑动窗口，只保留最近N轮
-  VectorStoreRetrieverMemory ：长时向量记忆，语义召回历史
 
记忆是注入到prompt里，作为上下文给到Chain
3. Context Compression 上下文压缩
-  ContextualCompressionRetriever ：检索出来的chunk，经过压缩器过滤/精简
- 压缩器：LLM压缩、Reranker过滤，删掉无关片段，减少token
 
二、其他配套生态组件（顺带）
 
- PromptTemplate / ChatPromptTemplate：提示模板，占位符填充
- OutputParser：解析模型输出，Pydantic输出结构化数据
- DocumentLoader：文档加载（PDF/Markdown/网页）
- TextSplitter：文本切分
- Embeddings：文本向量化封装
- VectorStores：向量库统一抽象层（Chroma、Qdrant、Milvus一套接口）
 
三、Chain vs Agent vs LangGraph 区分
 
1. Chain：固定流程，预先写死步骤，不会自主选择工具、不会循环思考；适合RAG、摘要、翻译这类确定性流程
2. Agent（LangChain旧Agent）：基于ReAct，模型动态选工具、动态决定下一步；早期实现，复杂分支难控制
3. LangGraph：LangChain生态的图框架，新一代Agent标准，自定义节点、边、条件分支、循环、断点，实现ReAct、规划、记忆、重规划
 
一句话：简单固定RAG用Chain；需要动态工具循环、复杂Agent，优先LangGraph
 
四、工程踩坑清单
1. Chain 耦合高：顺序写死，需求一变要大量改动，不要强行用Chain做复杂Agent
2. Memory容易混淆：Memory是给Chain用的；LangGraph一般自己维护state状态，不再依赖老Memory组件
3. 工具描述很关键：描述写模糊，Agent选错tool；参数schema尽量用Pydantic强约束
4. 上下文管理不要只依赖Memory：长对话必须搭配检索记忆+摘要，不要无脑塞全部历史
5. 老版本LangChain很多API废弃：现在推荐  langchain-core （基础抽象）、 langchain-community （第三方集成）、 langgraph 
 
五、 
- Chains：固定流水线，一步一步按预设跑
- Tools：外部能力单元，给模型调用
- Context Management：对话历史 + 记忆 + 上下文压缩，管好窗口里面装什么




LangGraph （基于状态机的复杂Agent）
 
定位：LangGraph 是 LangChain 生态里的状态图编排框架，专门解决传统 Chain 只能线性跑、旧版Agent难管控循环/分支/断点的痛点；用来落地 ReAct、任务规划、多Agent、重规划、记忆回退。
核心思想：Agent = 图，节点是动作，边控制流转，State（状态）是全局唯一的信息载体。
 
一、核心四大基础概念
1. State 状态
贯穿整个图的共享数据，所有节点读写都基于State，相当于Agent的全局上下文。
- 一般用  TypedDict  或者  Pydantic  定义字段：messages、plan、tool_results、error、next_step 等
- 内置消息列表  messages  最常用，保存用户消息、LLM输出、工具返回Observation
- Reducer（归约函数）：重点！同一个字段多节点并发写入时，定义合并规则；messages默认reducer是追加，而不是覆盖。
- 区分：State ≠ 短时记忆，State是图运行时状态；记忆模块可以把State内容持久化到向量库/数据库（长时记忆）
2. Node 节点
图里每一个执行单元，就是一个函数。输入State，返回State更新内容。
常见节点：
- llm_node：调用LLM，做思考、生成tool_call、生成计划
- tool_node：解析tool_call，批量执行工具
- planner_node：任务拆解规划
- summarizer_node：上下文摘要、压缩
- human_node：人工介入审批（Human-in-the-loop）
 
节点是纯函数：输入state → 返回要更新的字段，不直接修改原始state
3. Edge 边
定义节点之间的流转关系，两种：
- 普通边（Normal Edge）：固定流转，A执行完一定走到B
- 条件边（Conditional Edge）：根据当前State判断走向，是实现ReAct循环的关键
例：LLM输出是tool_call → 走到tool节点；否则 → 结束图，返回答案
4. Graph（StateGraph）
状态图本体：
python
  
from langgraph.graph import StateGraph
graph_builder = StateGraph(MyState)
 
 
工作模式：从  START  入口节点开始，沿着边流转，走到  END  终止。
 
二、ReAct 经典图结构（最常用模板）
 
节点： llm  →  tools 
条件边：llm执行完，判断输出是否包含工具调用
 
- ✅ 有tool_call → 流向 tools 节点
- ❌ 无tool_call → 流向 END，输出答案
tools执行完成后，固定边回到 llm 节点，形成循环
 
这就是标准 ReAct：Thought(llm) → Action(tools) → Observation(写入state) → Thought(llm)
 
plaintext
  
START → LLM → {判断：是否调用工具}
        ↓是        ↓否
       TOOLS       END
        ↓
       LLM（循环）
 
 
三、高级能力（LangGraph独有的，普通ReAct很难实现）
1. Human-in-the-loop 人工介入
断点机制：在指定节点设置  interrupt_before / interrupt_after ，图运行到这里暂停，等待人工输入、审批、修改计划后再继续执行。
适用：高危工具、重要决策、需要人确认的流程。
2. Checkpointer 持久化检查点
搭配  MemorySaver  / Redis / Postgres 存储每一步的state快照。
- 作用：暂停、恢复、回溯历史状态（回退到上一步、重做）
- 适合：长耗时任务、异常中断恢复、多轮会话Agent
 
区分：Checkpointer 存图运行快照，和业务长时向量记忆是两套东西
3. 并行节点（Parallel Branch）
同一个状态触发多个节点并行执行（同时调用多个检索、多个工具），全部完成后，reducer合并结果再往下走。
适合多源并行检索、多模型打分。
4. 子图 Subgraph
把一套完整图封装成一个节点，实现嵌套：主图负责调度，子图负责一个子任务（比如子RAG、子规划Agent），实现模块化。
 
四、和传统组件对比
- Chain：固定线性流程，没有循环、没有分支，无状态
- 老版 LangChain Agent：黑盒循环，流转不可见，很难插入自定义节点、断点、人工审核
- LangGraph：显式状态+显式流转，每一步可控，日志、观测、回退都方便
 
五、工程高频字段与模式
标准State示例（TypedDict）
 
python
  
from typing import Annotated, TypedDict, List
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages] # add_messages就是消息追加reducer
    plan: List[str]
    tool_results: List[dict]
 
-  add_messages ：LangGraph内置reducer，新消息追加到列表，不会覆盖历史对话
 
六、常见坑
1. 忘记reducer：直接覆盖messages，历史对话丢失
2. 循环没有终止条件：死循环一直调用工具（必须设置最大迭代次数）
3. State放太多冗余字段：状态膨胀，checkpoint存储开销大
4. 混淆 Checkpointer 和业务记忆：Checkpointer 是运行快照；向量记忆是业务知识库/用户历史
5. 节点里面写大量业务逻辑：节点尽量轻量化，逻辑抽离到外部函数
6. 条件分支判断逻辑复杂，LLM输出不稳定：尽量让LLM输出结构化tool_call，由代码做路由判断，不要让LLM自己决定分支
 
七、适用场景
✅ ReAct工具循环Agent、带重规划的任务Agent、多Agent协作、需要人工审批流程、长任务断点续跑、复杂RAG流水线
❌ 简单单步RAG、一次性Prompt调用（直接Chain就足够，没必要上LangGraph）
 
八、 
LangGraph = 状态机编排Agent
State是全局数据；Node是动作；Edge控制流转；条件边实现ReAct循环；Checkpointer做快照暂停与回溯。
 



CrewAI （角色扮演式多智能体协作）
定位：CrewAI 是面向多Agent分工协作的框架，主打角色+目标+任务的范式，和 LangGraph 状态机编排思路不一样。LangGraph 偏向灵活状态流转；CrewAI 天然适合模拟团队，给每个Agent分配岗位，分工配合完成复杂项目（写报告、市场调研、数据分析）。
 
一、核心四大基础概念
1. Agent（智能体，角色）
每个Agent是一个独立角色，拥有人设、目标、背景、可用工具。
 
-  role ：岗位角色（市场研究员、技术文档工程师、审核员）
-  goal ：这个角色要达成的目标
-  backstory ：背景人设，用来塑造风格、立场、能力边界（角色扮演核心）
-  tools ：该角色能调用的工具，不同角色工具可以不一样
-  llm ：可单独给每个Agent绑定不同大模型（研究员用强检索模型，写作用侧重创作模型）
-  max_iter ：单个Agent最大迭代轮次，防止死循环
 
重点：角色隔离！研究员只管收集信息，不负责润色；写手只管整理输出，不做检索。
 
2. Task（任务）
分配给Agent的具体工作单元，一个Crew由多个Task组成。
 
-  description ：任务详细描述，输入要求
-  expected_output ：明确输出格式、标准（非常关键，减少幻觉）
-  agent ：绑定由谁来做
-  dependencies ：依赖其他任务，等前置任务完成拿到结果再执行
-  output_file ：直接把结果保存到文件（报告类场景高频）
 
3. Crew（团队）
把Agent和Task组装成一个团队，定义执行流程。
 
python
  
crew = Crew(agents=[agent1, agent2], tasks=[task1, task2])
crew.kickoff() # 启动运行
 
 
调度模式两种：
 
- Sequential（串行，默认）：按任务依赖顺序依次执行，A任务输出作为B任务输入
- Hierarchical（分层主管模式）：自动生成一个主管Agent，负责分配任务、校验结果、协调冲突，适合任务多、依赖复杂的场景
 
4. Process（执行流程）
 
-  Process.sequential ：串行流水线（最常用、稳定）
-  Process.hierarchical ：分层主管调度，主管拆解、派活、汇总、把关
 
二、和普通单Agent、LangGraph的区别
 
- 单Agent（ReAct）：一个大脑，自己规划、自己调用工具、自己干活
- CrewAI：多个有独立人设的专家，各司其职，任务按流水线交付，天然适合文档产出类工作
- LangGraph：底层状态机，自由度极高，循环、分支、并行、断点都能做；偏向控制流
- CrewAI：上层应用框架，抽象是「团队分工」，控制流封装好了，上手快，但自定义流转灵活性弱于LangGraph
 
一句话区分：
你要精细控制每一步状态、分支、人工断点 → LangGraph
你想快速搭一个专家团队，比如调研→写稿→审核→排版 → CrewAI
 
三、典型工作链路（串行）
 
1. Task1（研究员Agent）：检索资料、收集信息 → 输出调研草稿
2. Task2（撰稿Agent）：读取Task1结果，整理成正式文稿
3. Task3（审核Agent）：校验内容、检查事实、修正错误
4. 全部完成，汇总最终交付物
 
后一个Task自动接收前面Task的输出作为上下文，不需要手动维护State。
 
四、高级特性
 
1. Agent间回调/协作消息传递
任务产出自动传递下游Agent；角色之间可以互相提问、核对信息。
2. 缓存Cache
工具调用、LLM推理结果缓存，减少重复token消耗
3. 异步kickoff
 kickoff_async() ，适合后台跑长时间报告任务
4. 回调Callbacks
监听每一步Agent思考、工具调用、任务完成事件，用于日志、监控
5. 自定义工具共享/隔离
工具可以全局共享，也可以只给指定角色，控制权限
 
五、Prompt设计核心要点（CrewAI成败关键）
 
1. backstory不要太长，聚焦岗位能力和约束；角色职责一定要划清，避免职责重叠抢活
2. expected_output 必须写清楚格式：markdown、json、要点列表
3. 明确禁止越权：研究员不允许直接下结论、写总结
4. 依赖任务要写明：参考【上一个任务的输出】
 
六、高频坑点
1. 职责模糊：多个Agent做一样的事，内容重复、互相矛盾
2. 缺少审核角色：产出内容事实错误没人校验，幻觉直接流出
3. 任务依赖写反，串行顺序错乱
4. 大任务粒度太大：一个Task同时要求检索+写稿+排版，失去分工意义
5. hierarchical模式主管幻觉：主管错误合并结果、丢失关键信息
6. 没有限制max_iter：某个Agent反复调用工具卡死
 
七、适用 / 不适用场景
✅ 适合：市场调研报告、竞品分析、技术文档撰写、论文辅助、方案策划、多步骤内容生产
❌ 不适合：动态强交互、频繁分支跳转、实时问答、需要大量人工介入审批的Agent
 
八、极简示例伪代码
python
  
from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="信息研究员",
    goal="收集准确资料",
    backstory="你是专业调研人员，只负责检索和整理原始事实",
    tools=[search_tool]
)
writer = Agent(
    role="撰稿师",
    goal="根据调研结果写成报告",
    backstory="擅长结构化撰写，不编造事实"
)

task1 = Task(description="检索xxx信息", expected_output="要点清单", agent=researcher)
task2 = Task(description="基于调研清单写报告", expected_output="markdown报告", agent=writer, dependencies=[task1])

crew = Crew(agents=[researcher, writer], tasks=[task1, task2], process=Process.sequential)
result = crew.kickoff()
 
九、 
CrewAI = 角色扮演多专家团队
Agent是岗位角色，Task是分配的工作；支持串行/主管分层调度；适合内容生产类多Agent项目；灵活性低于LangGraph，开发速度更快。
 




Dify（开源知识库）
定位：Dify 是开源低代码AI应用平台，核心是可视化RAG知识库 + 对话应用 + Workflow/Chatflow + Agent；不用从零写RAG链路，私有化部署，模型无关，适合快速落地企业知识库问答、业务Agent。
 
一、知识库核心链路（Dify RAG）
 
完整流水线：数据源接入 → 文档解析 → 文本分块Chunk → Embedding向量化 → 存入向量库 → 检索 → 重排 → 注入Prompt → LLM生成
 
1. 数据源接入
- 本地上传：PDF、docx、md、txt、csv、html；扫描件需要提前OCR
- 同步源：网页爬虫、Notion、飞书、语雀等在线文档
- Knowledge Pipeline（新版）：可视化编排ETL管线，清洗、抽取、过滤，可调试、可复用，适合企业批量文档处理 
2. 分块配置（重点调参）
- 分块大小：中文推荐 500~1000 token
- 重叠大小：50~100 token，防止上下文断裂
- 分隔符：按段落、标题分割；支持自定义分割规则
3. 向量库支持
内置轻量向量库；外接：PgVector、Qdrant、Milvus、Weaviate、Pinecone
4. 检索策略（知识库独立配置）
- 语义检索：Embedding余弦相似度
- 关键词检索：BM25
- 混合检索（Hybrid）：语义+关键词，Dify内置支持
- Reranker重排：接入BGE、Jina等重排模型，过滤召回无关片段
- 召回数量、分数阈值、元数据过滤（标签、时间）均可配置
5. 引用溯源：回答自带文档来源引用，方便校验幻觉
 
二、四大应用类型
1. 基础对话应用（Chatbot）
最简单，绑定知识库、系统提示词、模型，直接问答；适合普通知识库客服。
2. Chatflow（对话式工作流）
带对话记忆、多轮交互的可视化流程，适合多分支对话、表单收集、引导式问答。
3. Workflow（一次性工作流）
无对话记忆，单次触发执行；适合批量摘要、文档解析、数据处理、后台任务；支持API/Webhook/定时触发。
4. Agent应用
内置ReAct循环、工具调用、知识库检索；支持Function Calling、自定义工具、MCP工具；可以作为节点嵌入Workflow里使用 
 
三、Workflow常用节点
- LLM节点：调用大模型
- 知识检索节点：读取知识库（最常用）
- 条件分支If/Else、迭代循环节点
- 代码节点：Python/JS沙箱，做数据转换
- HTTP请求：调用外部API
- 人工审核Human Input：暂停等待审批，Human-in-the-loop
- 工具节点、Agent节点、变量模板节点 
 
四、模型与权限
- 模型无关：OpenAI、DeepSeek、通义千问、Qwen、本地vLLM/Ollama模型都可以接入
- 支持单独配置Embedding模型、Reranker模型、LLM模型分开选型
- 团队空间：成员、角色权限、知识库隔离、应用隔离
- 发布方式：网页嵌入、API接口、SDK、分享链接
 
五、记忆能力
- 对话记忆：内置会话历史，支持窗口大小、摘要压缩
- 区分：知识库=长时静态知识（文档）；对话记忆=当前会话上下文
- 不自带原生向量长时记忆；需要用户历史记忆可以自己用Workflow+向量库节点实现
 
六、和其他框架对比
- Dify：产品化平台，面向业务人员/后端快速上线，可视化，开箱RAG、日志、溯源；代码量少，二次开发自由度有限
- LangChain：底层组件库，代码驱动，灵活；适合深度定制
- LangGraph：状态机编排，适合复杂Agent循环、断点、分支
- CrewAI：多Agent角色扮演团队
一句话：快速上线企业知识库问答优先Dify；要深度自定义Agent逻辑选LangGraph
 
七、优势
✅ 私有化部署，数据不出内网，企业合规友好
✅ RAG链路可视化，检索测试面板，调参直观，不用写向量检索代码
✅ 溯源引用、日志、用户反馈、耗时统计、可观测性内置
✅ 一套知识库可以被多个Chatbot/Workflow/Agent复用
✅ 支持混合检索、重排、Pipeline预处理，生产级能力齐全
 
八、坑点与局限
1. 复杂多模态文档（图表、公式）解析弱，需要前置OCR/VLM抽取
2. 自定义Agent逻辑不如LangGraph灵活，复杂状态流转很难做
3. 内置分块策略有限，超复杂文档清洗要靠Pipeline代码节点
4. 向量库扩容、大规模海量文档场景，外接专业向量库
5. Agent是封装好的ReAct，不支持自定义规划器、复杂记忆策略
6. 扫描PDF、图片文档本身不能直接看懂，Dify知识库默认只处理文本
 
九、 
Dify = 可视化RAG平台 + 低代码工作流；
知识库负责文档入库、向量化、混合检索重排；Chatflow/Workflow编排业务流程，内置Agent做工具调用；适合快速交付企业知识库问答。
 


Dify （开源知识库）
定位：Dify 是开源低代码AI应用平台，核心是可视化RAG知识库 + 对话应用 + Workflow/Chatflow + Agent；不用从零写RAG链路，私有化部署，模型无关，适合快速落地企业知识库问答、业务Agent。
一、知识库核心链路（Dify RAG）
 
完整流水线：数据源接入 → 文档解析 → 文本分块Chunk → Embedding向量化 → 存入向量库 → 检索 → 重排 → 注入Prompt → LLM生成
 
1. 数据源接入
- 本地上传：PDF、docx、md、txt、csv、html；扫描件需要提前OCR
- 同步源：网页爬虫、Notion、飞书、语雀等在线文档
- Knowledge Pipeline（新版）：可视化编排ETL管线，清洗、抽取、过滤，可调试、可复用，适合企业批量文档处理 
2. 分块配置（重点调参）
- 分块大小：中文推荐 500~1000 token
- 重叠大小：50~100 token，防止上下文断裂
- 分隔符：按段落、标题分割；支持自定义分割规则
3. 向量库支持
内置轻量向量库；外接：PgVector、Qdrant、Milvus、Weaviate、Pinecone
4. 检索策略（知识库独立配置）
- 语义检索：Embedding余弦相似度
- 关键词检索：BM25
- 混合检索（Hybrid）：语义+关键词，Dify内置支持
- Reranker重排：接入BGE、Jina等重排模型，过滤召回无关片段
- 召回数量、分数阈值、元数据过滤（标签、时间）均可配置
5. 引用溯源：回答自带文档来源引用，方便校验幻觉
 
二、四大应用类型
 
1. 基础对话应用（Chatbot）
最简单，绑定知识库、系统提示词、模型，直接问答；适合普通知识库客服。
2. Chatflow（对话式工作流）
带对话记忆、多轮交互的可视化流程，适合多分支对话、表单收集、引导式问答。
3. Workflow（一次性工作流）
无对话记忆，单次触发执行；适合批量摘要、文档解析、数据处理、后台任务；支持API/Webhook/定时触发。
4. Agent应用
内置ReAct循环、工具调用、知识库检索；支持Function Calling、自定义工具、MCP工具；可以作为节点嵌入Workflow里使用 
 
三、Workflow常用节点
 
- LLM节点：调用大模型
- 知识检索节点：读取知识库（最常用）
- 条件分支If/Else、迭代循环节点
- 代码节点：Python/JS沙箱，做数据转换
- HTTP请求：调用外部API
- 人工审核Human Input：暂停等待审批，Human-in-the-loop
- 工具节点、Agent节点、变量模板节点 
 
四、模型与权限
 
- 模型无关：OpenAI、DeepSeek、通义千问、Qwen、本地vLLM/Ollama模型都可以接入
- 支持单独配置Embedding模型、Reranker模型、LLM模型分开选型
- 团队空间：成员、角色权限、知识库隔离、应用隔离
- 发布方式：网页嵌入、API接口、SDK、分享链接
 
五、记忆能力
 
- 对话记忆：内置会话历史，支持窗口大小、摘要压缩
- 区分：知识库=长时静态知识（文档）；对话记忆=当前会话上下文
- 不自带原生向量长时记忆；需要用户历史记忆可以自己用Workflow+向量库节点实现
 
六、和其他框架对比（高频考点）
 
- Dify：产品化平台，面向业务人员/后端快速上线，可视化，开箱RAG、日志、溯源；代码量少，二次开发自由度有限
- LangChain：底层组件库，代码驱动，灵活；适合深度定制
- LangGraph：状态机编排，适合复杂Agent循环、断点、分支
- CrewAI：多Agent角色扮演团队
 
一句话：快速上线企业知识库问答优先Dify；要深度自定义Agent逻辑选LangGraph
 
七、优势
 
✅ 私有化部署，数据不出内网，企业合规友好
✅ RAG链路可视化，检索测试面板，调参直观，不用写向量检索代码
✅ 溯源引用、日志、用户反馈、耗时统计、可观测性内置
✅ 一套知识库可以被多个Chatbot/Workflow/Agent复用
✅ 支持混合检索、重排、Pipeline预处理，生产级能力齐全
 
八、坑点与局限
1. 复杂多模态文档（图表、公式）解析弱，需要前置OCR/VLM抽取
2. 自定义Agent逻辑不如LangGraph灵活，复杂状态流转很难做
3. 内置分块策略有限，超复杂文档清洗要靠Pipeline代码节点
4. 向量库扩容、大规模海量文档场景，外接专业向量库
5. Agent是封装好的ReAct，不支持自定义规划器、复杂记忆策略
6. 扫描PDF、图片文档本身不能直接看懂，Dify知识库默认只处理文本
 
九、
Dify = 可视化RAG平台 + 低代码工作流；
知识库负责文档入库、向量化、混合检索重排；Chatflow/Workflow编排业务流程，内置Agent做工具调用；适合快速交付企业知识库问答。
 

Hermes Agent 
定位：函数调用驱动、轻量型Agent执行器，主打简洁、原生工具调用，不绑定复杂大框架
 
1. 核心概念
 
Hermes Agent 源自 Hermes 系列模型生态（Hermes LLM），配套的轻量Agent运行组件，核心依赖原生Function Calling，不自带复杂状态图、记忆管理、多Agent编排，区别于LangGraph、CrewAI。
 
- 设计目标：极简，把「思考→选工具→调用工具→带回结果→继续推理」闭环跑通
- 核心驱动：LLM输出标准function call，执行器负责解析、调用、回传
 
2. 基础执行流程（单轮循环）
 
1. 接收用户query + 工具schema列表
2. LLM判断：直接回答 / 需要调用工具
3. 输出结构化Function Call（工具名、参数）
4. Hermes执行器校验参数、执行对应工具函数
5. 获取工具返回结果，拼接进上下文
6. 再次交给LLM，继续推理，直到不需要工具，输出最终答案
 
3. 关键特性
 
- ✅ 轻量：无 heavy 依赖，代码量小，适合嵌入服务
- ✅ 强绑定标准Function Call：依赖模型输出规范JSON工具调用，不依赖prompt模拟工具调用
- ✅ 可自定义工具：注册普通Python函数，自动转schema（类似OpenAI工具定义）
- ✅ 基础循环控制：支持最大迭代轮次，防止无限调用工具
- ✅ 可扩展记忆：本身不带内置记忆，短时/长时记忆需要外部自己接入（向量库、消息队列、数据库）
- ✅ 无内置分支图：不像LangGraph支持自定义条件分支、多条并行路径，默认是线性ReAct循环
 
4. 和同类框架对比
 
- Hermes Agent：轻量执行器，只做 function call 调度，适合简单工具Agent、快速上线
- LangGraph：状态机驱动，支持复杂分支、断点、人工介入，适合复杂业务Agent
- CrewAI：多Agent、角色分工、任务委派，面向多智能体协作
- Dify：低代码平台化，可视化编排，开箱RAG+Agent
 
5. 适用场景
 
✅ 简单工具查询Agent：查天气、查数据库、接口查询、计算器
✅ 内部轻量服务嵌入，不想引入LangChain整套庞大生态
✅ 基于Hermes系列模型快速搭建工具调用能力
❌ 不适合：复杂分支逻辑、多Agent协作、长时记忆复杂管理、需要人工审批断点的流程
 
6. 工程坑点
 
1. 模型输出function call格式异常：需要加重试、JSON容错解析
2. 循环爆炸：必须设置 max_steps 最大调用次数上限
3. 工具调用权限隔离：执行器要做参数校验、沙箱，防止恶意调用
4. 本身无RAG：检索能力需要自己额外接入Embedding+向量库
5. 记忆裸奔：会话历史需要业务侧维护消息列表
 
7. 
Hermes Agent = 轻量Function Call调度器，线性ReAct循环，专注工具执行，复杂能力全部外部扩展



 了解框架LlamaIndex + AutoGen 
定位：两大主流Agent/RAG框架，侧重不一样；LlamaIndex主打数据检索RAG，AutoGen主打多智能体对话协作
一、LlamaIndex（原名 GPT Index）
 
核心定位
 
以数据、检索、索引为核心的框架，强项是RAG，解决“大模型怎么和私有数据打交道”。
 
核心组件
 
1. Connector/Reader：文档加载器（PDF、Markdown、网页、数据库等）
2. Node：文档切片单元（相当于chunk，比原始文档更小的处理单位）
3. Index 索引：核心存储抽象
- VectorStoreIndex：向量检索（最常用）
- SummaryIndex：摘要检索，全文概括
- KeywordTableIndex：关键词检索
- ListIndex：顺序遍历检索
4. Retriever 检索器：从索引里召回相关节点，支持混合检索、重排
5. QueryEngine 查询引擎：检索+LLM生成，基础问答入口
6. ChatEngine：带会话记忆的对话问答
7. Tools / Agent：把检索索引包装成工具，供Agent调用
 
特点
 
- 优势：RAG链路高度封装，大量内置加载器、检索策略、后处理（重排、上下文压缩），开箱做知识库
- 弱项：原生多Agent编排能力弱，Agent是附加能力，不是主线
- 适配场景：私有知识库、文档问答、企业RAG系统
- 底层：可对接任意LLM、Embedding、向量库（OpenAI、DeepSeek、Milvus、Qdrant、Chroma等）
 
简单流程
 
文档加载 → 切分Node → 向量化 → 构建索引 → 检索召回 → 传入LLM生成答案
 
二、AutoGen（微软）
 
核心定位
 
多Agent协作框架，主打多个智能体互相对话、分工协商完成任务，RAG属于外接能力。
 
核心概念
 
1. Agent 智能体
- AssistantAgent：普通LLM助手，写代码、推理
- UserProxyAgent：模拟人类，负责接收结果、执行代码、反馈
- GroupChat / GroupChatManager：多Agent群聊管理器，负责选谁发言、流转对话
2. 对话循环：Agent之间互相发消息，自动来回沟通，直到任务完成
3. Code Executor 代码执行器：内置沙箱执行Python代码（数据分析、计算、脚本），是AutoGen王牌能力
4. Function Calling：支持工具调用，可自定义函数
5. 可接入记忆、RAG工具，但是框架本身不提供索引能力
 
特点
 
- 优势：多Agent对话编排成熟、代码执行能力强，适合需要分工、互相评审、写代码调试的复杂任务
- 弱项：没有内置RAG索引链路，文档检索需要外接LlamaIndex / LangChain
- 适配场景：代码助手、数据分析Agent、多角色协作任务、复杂问题多方讨论
 
简单流程
 
用户任务 → 多个Agent加入群聊 → 轮流对话、调用工具、运行代码 → 互相校验反馈 → 产出最终结果
 
三、横向对比（LlamaIndex / AutoGen / LangGraph / CrewAI）
 
- LlamaIndex：RAG优先，擅长文档检索问答，单Agent为主
- AutoGen：多Agent对话优先，擅长群聊协作+代码执行
- LangGraph：状态机优先，自定义DAG流程、断点、人工介入，灵活度最高
- CrewAI：角色任务优先，按任务清单分配角色，偏向固定流水线多Agent
 
四、工程坑点
 
1. LlamaIndex
- 版本改动大，新旧API不兼容
- 默认切片策略简单，复杂文档要自己优化chunk、重叠度
- Agent能力偏弱，复杂工具链建议搭配LangGraph
2. AutoGen
- 多Agent容易无限闲聊、循环对话，必须设置终止条件
- 代码执行有安全风险，生产环境严格隔离沙箱
- 没有内置知识库，RAG需要集成LlamaIndex
 
五、
- LlamaIndex：RAG专用框架，让大模型读懂私有文档
- AutoGen：多Agent群聊框架，让多个AI互相分工对话、写代码
 
