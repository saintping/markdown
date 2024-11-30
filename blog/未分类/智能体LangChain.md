### 前言
以OpenAI为代表的大语言模型成爆发之势，在垂直领域落地还有两个问题需要解决。AI Agent（智能体）给出的方案分别是：

- 内容幻觉
  大模型没有训练到的数据怎么办？大模型的训练数据都来自于互联网某个时间点的快照。企业内部数据大模型无法访问，快照点之后的数据对大模型也是未知。在没有内容支撑时，大模型很容易说一大通空话和废话。可以使用RAG技术Retrieval-Augmented Generation（索引增强生成）让大模型也知道和理解这些数据。
- 态势感知
  有点像P2P网络中的预言机，只是Agent的Tools不仅支持信息的输入，也支持执行的下发。

### LLM Agent架构

![llm-agent.png](https://ping666.com/wp-content/uploads/2024/11/llm-agent.png "llm-agent.png")

以上就是智能体的一个大致结构，典型的Re-Act模型。大模型负责推理，Tools负责具体的执行。

LangChain是一个开源的大模型应用框架。
[https://github.com/langchain-ai/langchain.git](https://github.com/langchain-ai/langchain.git "https://github.com/langchain-ai/langchain.git")

核心理念是chain，将各种动作通过人工编码成chain后去执行。
>LangChain is a framework for developing applications powered by large language models (LLMs).

![langchain_stack.jpg](https://ping666.com/wp-content/uploads/2024/11/langchain_stack.jpg "langchain_stack.jpg")

### LangChain使用
LangChain是一个Python项目，部署非常简单。
`pip install langchain`

第一个LangChain程序
```python
from langchain_community.chat_models import ChatZhipuAI
from langchain_core.messages import HumanMessage, SystemMessage

chat = ChatZhipuAI(
    model="glm-4-plus",
    temperature=0.5,
)

messages = [
    SystemMessage(content="你是个诗人"),
    HumanMessage(content="用一首绝句夸一夸智谱GLM大模型很厉害"),
]

response = chat.invoke(messages)
print(response.content)
```
>智谱GLM显神通，
海量知识一网中。
妙笔生花文采溢，
思维敏捷胜人工。

### Chain VS Agent
写一个股票助手的应用。分别使用Chain和Agent方式。

##### Agent
- 代码
```python
# langchain.debug = True
pythonREPLTool = PythonREPLTool()
wikipedia = WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())
search = TavilySearchResults(max_results=3)
api_wrapper = FinancialDatasetsAPIWrapper(
    financial_datasets_api_key=os.environ["FINANCIAL_DATASETS_API_KEY"]
)
financial_toolkit = FinancialDatasetsToolkit(api_wrapper=api_wrapper)
tools = financial_toolkit.get_tools() + [pythonREPLTool, wikipedia, search]
memory = SimpleMemory()
model = ChatZhipuAI(
    name="股票助手",
    model="glm-4-plus",
    temperature=0.5,  # 一般设置在[0, 1.0]，温度越高，随机性越高
    max_tokens=10 * 1024,  # 最大输出token
)
prompt = hub.pull("hwchase17/structured-chat-agent")  # 每个agent有他对应的默认提示语
agent = create_structured_chat_agent(model, tools, prompt)
agent_executor = AgentExecutor(
    agent=agent, tools=tools, memory=memory, handle_parsing_errors=True, verbose=True
)
agent_executor.invoke(
    {"input": "今天是2024年11月11号，苹果股票还值得投资吗？请给出具体的3条理由。"}
)
```

- 输出
>{'input': '今天是2024年11月11号，苹果股票还值得投资吗？并且给出数据理由。',
 'output': 'Based on the latest financial data, Apple Inc. (AAPL) shows strong profitability and operational efficiency, with a return on assets of 25.68% and an operating cash flow margin of 30.24%. However, its high debt-to-equity ratio (5.41) and current ratio (0.87) suggest some financial risks. Overall, Apple remains a potentially worthy investment, but caution is advised due to its debt levels and liquidity position. Consider market conditions and your investment strategy before deciding.'}

- 推理过程
这里给他配置了一系列的工具，用于计算的pythonREPLTool，用于搜索的wikipedia和search，用于股票分析的财经工具集合financial_toolkit。
关键是提示语prompt的内容"hwchase17/structured-chat-agent"。告诉大模型，一遍一遍使用所需的工具去查找支撑数据，然后分析出最终结果。
![prompt.png](https://ping666.com/wp-content/uploads/2024/11/prompt.png "prompt.png")
智能体大致分析过程就是这样的。但是智谱GML看起来不太聪明的样子。O(∩_∩)O

##### Chain
基于大模型在推理环节不太靠谱，其实可以人工把股票的分析过程，硬编码成chain让大模型去执行就好了。这也是为什么各行各业冒出大量智能体的底层原因。
这时大模型就只负责他擅长的环节：NLP/语音处理，程序员硬编码负责推理过程。真的人工+智能。
