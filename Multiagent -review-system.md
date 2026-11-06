# PR Review Agent 设计研讨纪要（LangGraph 方案）
> 背景：面向 Azure DevOps Git 仓库的代码 PR 智能评审 Agent，基于 LangGraph 搭建；支持动态 Skill 选择、多文件跨文件依赖检查、轻量 AST 语义辅助，适配 Gemini 3.8 Flash / Gemini 2.5 Pro。

## 一、总体定位
Agent 职责：**只做代码评审，不充当 PR 数据库**。
- 不开放按大范围时间批量扫描 PR 的入口，避免资源爆炸、状态膨胀。
- 输入约束：单次仅支持同一个 repo，传入 PR ID 列表。
- 能力扩展：预留 raw diff / commit hash 独立入口，用于 IDE pre-commit 本地预评审（还未提 PR 的阶段）。

## 二、接口定义
### 输入参数
| 参数 | 说明 | 约束 |
|---|---|---|
| repo | Azure Git 仓库名称 | 必填，一次请求只能一个仓库 |
| pr_id_list | PR ID 数组 | 上限 50 个 |
| model | 大模型 | 可选；默认：`gemini-3.8-flash`；可选 `gemini-2.5-pro` |
| confidence_threshold | 置信阈值 | 可选；默认 0.7；只返回分数 ≥ 阈值的问题 |

### 输出 JSON 结构
```json
{
  "request_id": "本次请求唯一ID",
  "model_used": "实际使用的模型",
  "confidence_threshold": "本次生效阈值",
  "pr_results": [
    {
      "pr_id": "PR编号",
      "repo": "仓库名",
      "comments": [
        {
          "file_path": "问题文件路径",
          "line_no": "问题行号",
          "skill_tag": "命中的skill标签",
          "problem_desc": "问题描述",
          "score": "置信度分数"
        }
      ]
    }
  ]
}
```

## 三、Skill 体系设计
- Skill 以独立 Markdown 文件存放，目录分层：
  - `common/`：通用评审规则
  - domain 目录：按技术领域划分（C++ / Python / 硬件寄存器等）
- 总量：30～50 个 skill md 文件
- 路由策略（Skill Router）：
  1. 预先为每个 skill md 提取一句简短描述（索引）
  2. 把 diff 特征 + AST 提取符号 + 全部 skill 摘要列表交给 LLM
  3. LLM 自主选出本次评审应该启用哪些 skill，返回 skill 文件名称列表
  4. 再加载对应完整 md 送入评审节点
> 优势：不需要硬编码规则；新增 skill 只需要放 md 文件，不用改路由代码。

## 四、LangGraph 节点流水线（执行顺序）
1. **PR 数据加载节点**
   - 拉取 PR diff + 上下文（3～5行 context）
   - 原始 diff 存入 Azure Blob；Graph 状态只存 blob 引用链接，不存大文本。
2. **轻量 AST 解析节点**
   - 解析 diff 改动位置：函数、结构体、变量符号
   - 按需从 Azure Repo 拉取符号定义（函数签名、结构体字段），生成结构化语义信息写入状态
   - 设计原则：**轻量 AST，不是全文件静态分析**，控制开销。
3. **Skill 路由选择节点**
   - 读取 skill 索引，交给 LLM 匹配，输出本次选中 skill 清单
4. **动态并行评审节点**
   - 根据选中 skill 列表，动态并行拉起多组评审分支
   - 每个分支加载对应 skill md，做单文件 diff 评审
   - 输出：单文件问题摘要、跨文件关联符号清单
5. **关联调用补全节点**
   - 根据改动符号，检索仓库调用点；拉取调用位置短上下文
   - 解决痛点：函数签名变更，但调用点不在本次 diff 内，diff 看不见的漏改问题
6. **跨文件聚合评审节点**
   - 汇总单文件评审结果 + 补充的调用上下文，做跨文件逻辑一致性校验
7. **整合校验节点**
   - 结果去重、按置信阈值过滤，组装成标准输出结构

## 五、LangGraph 基础模型（节点 / 边 / 路由 / Loop）
### 节点 Node
- 节点本质就是 Python 函数
- 输入：**全局 State（TypedDict）**
- 输出：更新后的 State 字典
- 节点只负责业务逻辑，**不负责自己决定下一跳往哪走**，逻辑和流转解耦。

### 边 Edge
两类边：
1. **固定边 add_edge(A, B)**：A 执行完固定走到 B，不需要路由函数
2. **条件边 add_conditional_edges**：需要单独写一个路由函数
    - 路由函数输入：同样是全局 State
    - 路由函数输出：下一跳节点名字符串
    - 一个源节点可以挂载独立的路由函数；不是全局只能一个路由函数

### Loop（循环）
- **不需要写 for / while**
- 循环 = 条件边回指到前面的节点
- 路由函数根据状态标记判断：继续循环，还是退出到下一个节点

### 图组装方式
- 使用 `StateGraph(StateType)` 作为图的构造器
- `.add_node(name, func)`：注册节点
- `.add_edge()` / `.add_conditional_edges()`：注册流转关系
- `.compile()`：编译成可执行图实例

## 六、MemorySaver 状态持久化
- LangGraph `MemorySaver`，后端对接 Azure CosmosDB，分区键 `session_id`
- State 里面**只存轻量结构化数据**；大文本（diff、完整skill md）全部外置到 Azure Blob Storage
- 特性：自动持久化、断点续跑、历史会话回溯；业务代码不需要手写 CRUD
- 每次 API 请求生成唯一 `session_id`

## 七、并发与 Python GIL 问题
- LangGraph 并行节点默认使用线程池
- IO密集场景（调用Gemini API、Azure Repo拉代码、Blob读取）：GIL 在等待 IO 时释放，并行收益很好
- CPU密集场景（复杂AST、大量静态计算）：GIL 限制多核并行，需要切换进程池执行来绕开 GIL
- 本项目场景：绝大多数节点是 IO 密集（LLM调用、拉代码），线程并行是合适的

## 八、LangGraph 框架本质与 AI 专属能力
1. 底层本质：**通用有向图编排框架（支持有环、无环；串行、并行分支）**，理论上可以编排任意数据流，不局限 LLM。
2. 针对 LLM / Agent 额外内置 building block：
   - 消息状态、工具调用原生支持
   - MemorySaver 持久化体系
   - Human-in-the-loop（人工介入中断）
   - 子图（subgraph）复用能力
   - 重试、断点、流式输出、可视化调试
   - 原生对接 LangChain 模型、工具生态

## 九、风险与治理约束
1. 批量上限：单次最多 50 个 PR，禁止大范围时间窗口批量扫描
2. 上下文治理：两级评审策略（单文件评审 → 摘要聚合跨文件评审），按需拉取关联代码，禁止一次性灌入全仓库代码
3. 模型策略：默认 Flash 快速评审；跨文件深度校验场景可切换 2.5 Pro
4. Debug 扩展：可增加 debug 开关，输出低于阈值的候选告警，用于审计复盘
5. CI/CD 集成：评审结果可接入 Azure Pipeline，用作门禁、自动回写 PR 评论

## 十、和成品 Agent（豆包、Meta Muse）架构差异
- LangGraph：**Agent 开发脚手架**，底层图编排可控；节点、流转、状态、记忆策略全部自定义；适合企业内部私有化、强流程、强权限、强审计的定制 Agent。
- 豆包 / Muse：**成品封装好的 Agent 产品**，对外只有对话接口；内部执行链路、路由、记忆策略黑盒，无法深度定制流水线。

# 本地多组件服务开发与发布方案
## 一、核心组件清单
1. 带Gemini ACP封装的FastAPI服务
2. PR Review Agent
3. PR Metrics统计服务
4. 带数据库的Portal展示站
5. Review Skill知识库（md格式技能文件）

## 二、组件设计细节
### 1. 带Gemini ACP的FastAPI服务
- 封装Gemini ACP调用逻辑，对外提供统一接口
- 生产环境使用端口8085，开发环境使用8086，挂载对应环境独立Docker网络
- 随版本镜像打包，通过根目录release.toml锁定运行版本

### 2. Review Skill知识库
- 初始阶段仅面向PR Review Agent使用，采用静态md文件直读，将文件读取逻辑封装为独立工具函数，避免业务代码散耦合
- 后续扩展多调用方时，可无缝替换为轻量REST接口，预留3个基础接口：GET /skills、GET /skills/{name}、GET /skills/{name}/version
- 生产环境只读release.toml锁定的提交版本md文件，开发环境加载最新md文件

### 3. Portal与数据库设计
- 生产与开发环境使用独立数据库存储目录，生产用./data/prod-db，开发用./data/dev-db，物理隔离schema与业务数据
- Git仅存储带版本号的数据库迁移SQL脚本，不保存真实业务数据文件
- 生产发版流程：先备份生产库，执行对应版本迁移脚本更新schema，校验通过后启动新版服务；开发环境启动时自动执行最新迁移脚本，用测试数据调试
- 业务数据通过定时任务备份至独立存储，不纳入Git管理

## 三、Docker环境与发布规则
### 1. 多环境网络隔离
- 创建prod-net与dev-net两个独立自定义Docker网络，无需集群管理员权限，本机Docker账号即可创建
- 生产组件全部接入prod-net，开发组件全部接入dev-net，两网络默认不通，避免跨环境误连
- 生产Portal使用8080端口，开发Portal使用8081端口，端口映射写在对应环境docker-compose配置中

### 2. 版本与镜像管理
- 单Git仓库管理所有组件代码，每个提交对应唯一哈希值，CI自动打包对应组件Docker镜像
- 镜像推送至Azure Container Registry私有仓，按组件名+Git短哈希+版本号打标签
- 根目录放置release.toml，正式发版时写入各组件固定镜像标签与版本号，启动脚本读取该文件拉取对应镜像

### 3. 启动脚本规则
- 根目录提供start-prod.sh与start-dev.sh两个一键启动脚本
- 脚本自动检测对应网络是否存在，不存在则自动创建，再加载对应环境compose配置与端口映射
- 开发环境启动脚本自动加载最新迁移脚本与md知识库，生产脚本严格遵循release.toml锁定版本

## 四、关键约束
1. 业务数据、数据库文件不纳入Git管理，仅迁移脚本与静态资源入库
2. 生产环境禁止直接使用latest镜像，所有版本由release.toml显式指定
3. 开发与生产环境容器、数据库、网络全隔离，避免schema冲突与数据污染

