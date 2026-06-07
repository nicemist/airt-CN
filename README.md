> 本项目为 [AIRT](https://github.com/0x4D31/airt) (AI Red Team Academy) 的中文翻译版本。旨在为国内安全研究员与从业者提供系统性、纯实战的 AI 安全学习路径，沉淀并攻克 AI 系统红队评估、提示词注入、RAG 投毒以及大模型基础设施安全等前沿攻防技术。
>
> 原项目由 0x4D31 维护。

# AIRT-CN — AI 红队学院中文版

这是一个免费且开源的课程，涵盖了针对 AI 系统的攻击性安全测试——从提示词注入到供应链攻击。包含 60 多个小时的高价值内容，并配备了基于 Docker 的本地实操实验室。

🌐 **[查看课程详情 →](https://0x4d31.github.io/airt/)**

## 课程模块

| # | 模块名称 | 核心技术覆盖点 / 主题 |
|---|--------|--------|
| 1 | AI 红队基础 | MITRE ATLAS 框架、OWASP LLM Top 10、威胁建模 |
| 2 | 提示词注入攻击 | 直接/间接提示词注入、越狱、过滤器绕过 |
| 3 | RAG 利用与向量数据库攻击 | 知识库投毒 、嵌入向量攻击 |
| 4 | 多智能体系统利用 | 智能体劫持 、工具/插件滥用、内存投毒 |
| 5 | AI 供应链与基础设施攻击 | 模型后门 、Pickle 反序列化漏洞利用、依赖项攻击 |
| 6 | 模型提取与推理攻击 | 模型窃取、成员推理攻击、侧信道攻击 |
| 7 | 大规模自动化 AI 红队演练 | garak、PyRIT、promptfoo 工具链使用，以及 CI/CD 流程集成 |
| 8 | 后渗透利用与影响分析 | 横向移动、报告编写、合规与监管框架 |

## 实操实验室

每个模块都包含一个基于 Docker 的独立实验环境。**无需任何云端 API 密钥**——所有大模型均通过 [Ollama](https://ollama.com/) 在本地环境运行。

### 快速开始

```bash
# 克隆本项目仓库
git clone https://github.com/nicemist/airt-CN
cd airt/labs

# 启动任意实验室（以 Lab 01 为例）
cd lab01-foundations
docker compose up

# 访问本地实验交互界面
http://localhost:8888
```

### 环境要求

- 已安装 Docker
- **内存**：8 GB+ RAM（运行实验 07–08 的大规模自动化工具时，推荐 16 GB）
- **硬盘空间**：需预留约 20 GB 空间，用于下载本地开源大模型

## License

Content is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). Code and lab files are licensed under [MIT](LICENSE).

