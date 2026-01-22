# MCP

**URL**: https://docs.dolphindb.cn/zh/mcp/mcp_introduction.html

**来源**: DolphinDB 官方文档

---

MCP
MCP（Model Context Protocol）是一种标准协议，用于将
大模型调用工具
的能力从原本紧耦合的 Function Call
            模式中解耦。在传统模式下，AI 工具调用是绑定在特定系统内，例如 ChatGPT 的函数调用只能使用它自己的插件。而 MCP
            的作用是
将“会话端”和“工具端”分离
：会话端只需实现 MCP 调用功能，具体的 MCP 工具可以在不同后端独立开发和维护。
在 DolphinDB 3.00.4 版本中，原生实现了 MCP server，支持一键将用户自定义函数转化成 MCP Tool，从而为 Agent 提供数据分析能力。
工具定义统一存储在支持 MCP Server 的 DolphinDB 控制节点上，执行时根据所连接的数据节点或计算节点执行函数逻辑。
会话界面（Agent 前端）可以是任何支持 MCP 协议的客户端，例如：VS Code Copilot, Claude, Cherry Studio,
                    DolphinDB 因子平台 AI 助手等。
大模型作为客户端的调度大脑，在会话过程中根据上下文决定用哪个工具以及填入哪些参数。
注意：
当前版本仅支持 MCP Tools 和 MCP Prompts。