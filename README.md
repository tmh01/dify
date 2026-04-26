# 📚 论文检阅 (AI Academic Assistant)

**论文检阅 (AI Academic Assistant)** 是一个基于 [Dify](https://dify.ai/) 平台构建的智能体工作流 (DSL)。

本项目能够根据用户的自然语言对话，自动分发至 **6 大专属处理分支**。从多源数据库的并行检索、相似文献交叉推荐、引文网络演进分析、全文本深度精读，到最终的 Zotero 自动化库管，为您提供一站式的 AI 科研辅助体验。

## 🌟 核心特性与架构 (The 6-Branch Architecture)

本项目基于底层的 6 个路由分支构建：

### 🔍 1. Search (全网综合检索)

- **参数级提取引擎**：自动将用户的模糊描述（如“让大模型变笨”）映射为专业英文学术术语（如 `capability control`, `activation engineering`）。
- **四源并发检索**：同时调用 Semantic Scholar, OpenAlex, arXiv, DBLP 四大学术库。
- **智能算分与重排 (Rank)**：内置 Python 算分引擎，结合 CCF 评级（内置权威字典）、被引频次对数平滑、年份红利，引入了随机微扰 (Random Jitter)，保障推荐的多样性。

### 🤝 2. Recommend (相似文献推荐)

- **核心实体抽取**：当用户提供一篇“种子论文”时，系统提取其最核心的 2-3 个技术实体（如 `Vision Transformer AND Hierarchical architecture`）。
- **跨域泛搜**：抛弃单纯依赖引用链的传统推荐，利用核心概念进行跨数据库“泛搜”，完美找寻底层技术思路线索相似的潜在文献。

### 🕸️ 3. Cite (引文寻迹与演进图谱)

- **宏观态势分析**：抓取被引记录，提取高频学者、核心流派词汇和年份分布。
- **演进综述生成**：撰写《核心文献引文与学术演进报告》。
- **JSONBin 动态图谱**：自动生成可视化网络图谱数据并托管，提供直达的知识网络可视化链接。

### 📖 4. Read (深度精读与解析)

- **开源 PDF 寻踪**：自动寻找论文的 OpenAccess 原文链接或 ar5iv 网页。
- **Jina Reader 穿透**：无视网页排版与 PDF 格式限制，清洗提取纯净的 Markdown 原文。
- **DeepSeek-Reasoner 深度思考**：将数万字长文本喂给推理模型，输出严谨的“方法论深度解析”、“实验与数据提炼”及“局限性分析”报告。

### 🧠 5. Analyze (应用文献分析)

- **Map-Reduce 并行架构**：利用 Dify 迭代节点的 `[并行模式]`，并发读取目标论文的几十篇参考文献。
- **全景综述归纳**：先由子 LLM 独立总结单篇，再由主 LLM 进行全局统筹，生成数据集创新方法表格。

### 💾 6. Save (Zotero 一键数字资产化)

- **隐形水印追踪**：工作流在输出结果时嵌入了隐藏的 `paperId` 注释。
- **云端直联入库**：当用户指示保存时，全自动装配标准 JSON Payload 并通过 Webhook 发送至个人的 Zotero 云端文献库，彻底告别手动录入。

## 🛠️ 技术栈与依赖 (Prerequisites)

导入本工作流 DSL 之前，请确保您准备了以下模型与 API Key：

1. **大语言模型引擎**：
   - 意图路由/快速总结节点：推荐 `DeepSeek-Chat`。
   - 深度长文本解析节点：**必须** `DeepSeek-Reasoner` 或同级别强推理模型。
2. **Jina Reader API** (`r.jina.ai`)：用于将 URL 转换为大模型友好的 Markdown。
3. **Zotero API Key & User ID**：用于激活 Save 自动化入库功能。
4. **JSONBin API Key**：用于激活引文知识图谱托管功能。
5. *可选*：**Semantic Scholar API Key**（免 Key 亦可使用，但配置 Key 可提高并发检索的稳定性与速率上限）。

## 🚀 部署与使用 (Getting Started)

### 第一步：导入工作流

1. 下载本项目中的 `dify-academic-workflow.yml` 文件。
2. 登录您的 Dify 工作台。
3. 点击 **"创建应用"** -> 选择 **"导入 DSL 文件"**。
4. 上传本 YAML 文件并确认。

### 第二步：配置您的私有环境变量

进入应用编排页面，找到相应的节点替换您的私人 Key：

- **`read` / `analyze` 分支**：在调用 `https://r.jina.ai/` 的 HTTP 节点 Header 中配置 `Authorization: Bearer <Your_Jina_Key>`。
- **`save` 分支**：在调用 Zotero API 的 HTTP 节点中，将 URL 路径中的 `User ID` 替换为您自己的，并在 Header 配置 `Zotero-API-Key: <Your_Zotero_Key>`。
- **全网搜索代码节点**：如果您有 Semantic Scholar API Key，可在 Python 代码节点的 Header 定义中加入 `x-api-key`。

### 第三步：开始对话！(Prompt Examples)

尝试向您的智能体发送以下指令：

> **全网检索**："帮我找找近两年关于 Mamba 处理序列矩阵指数化的顶级会议文章，最好是 CCF-A 类的。"
>
>
>
> **深度精读**："仔细阅读这篇论文的原文：`https://arxiv.org/abs/xxxx.xxxx`，告诉我它的核心创新机制和使用的数据集。"
>
>
>
> **引文寻迹**："分析一下上面那篇论文被谁引用了？生成它的技术演进趋势。"
>
>
>
> **一键保存**："把第二篇论文存到我的 Zotero 库里。"

## 🤝 贡献与参与 (Contributing)

学术开源，探索无界。非常欢迎提交 Issue 探讨检索算分算法的优化，或者提交 Pull Request 完善对更多学术数据库（如 IEEE Xplore, PubMed）的兼容与接入！