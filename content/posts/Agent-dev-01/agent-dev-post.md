+++
date = '2026-04-23T19:05:44+08:00'
draft = true
title = 'Agent Dev Post'
+++

## 环境搭建

### Python环境

python:`3.12.8`

### 创建虚拟环境并安装依赖

`<font style="color:rgb(54, 70, 78);background-color:rgb(245, 245, 245);">pip install uv</font>`<font style="color:rgb(54, 70, 78);background-color:rgb(245, 245, 245);">: 安装uv</font>

`uv --version`: 查看版本号

`uv init`+`uv venv`:<font style="color:rgb(15, 17, 21);">会自动创建虚拟环境（默认在 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">.venv</font>`<font style="color:rgb(15, 17, 21);"> 目录）并生成 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">pyproject.toml</font>`<font style="color:rgb(15, 17, 21);"> 和 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">uv.lock</font>`<font style="color:rgb(15, 17, 21);"> 文件</font>

### LLM-API

选择模型GLM [https://open.bigmodel.cn/](https://open.bigmodel.cn/)

<!-- 这是一张图片，ocr 内容为： -->

![](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776938958410-abf7c87a-8f92-4e05-a28f-87fe4e8ac4f4.png)

GLM提供新人免费额度

<!-- 这是一张图片，ocr 内容为： -->

![](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776939039905-33a00545-188d-4153-a05b-7151ce535c56.png)

## 工程实践

### 技术选型

基于 `**FastAPI + ChromaDB + LangChain + Streamlit**`的电商 RAG 问答入门学习项目。

FastAPI：<font style="color:rgba(0, 0, 0, 0.87);">一个现代、快速（高性能）的 Web 框架，用于基于标准 Python 类型提示构建 API。（</font>`**<font style="color:rgb(15, 17, 21);">FastAPI + Uvicorn(服务器)</font>**<font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);">类似于</font>`<font style="color:rgb(15, 17, 21);">SpringBoot(内嵌Tomcat、Jetty等)</font>`<font style="color:rgba(0, 0, 0, 0.87);">）</font>

**ChromaDB：**<font style="color:rgb(37, 41, 51);">一个开源的向量数据库，专门用于构建AI应用</font>

**LangChain：**<font style="color:rgb(60, 60, 67);">快速开始使用任何您选择的模型提供商来构建智能体</font>

**Streamlit： 一个用于机器学习、数据可视化的 Python 框架。**

* * *

`uv add langchain`

`**uv add langchain langchain-openai**`

### Demo-调用GLM-4.6V

    ZHIPUAI_API_KEY=Your_API_KEY_HERE
    
    import os
    
    from langchain_openai import ChatOpenAI
    
    api_key = os.getenv("ZHIPUAI_API_KEY")
    if not api_key:
        raise ValueError("请先设置环境变量 ZHIPUAI_API_KEY")
    
    llm = ChatOpenAI(
        model="GLM-4.6V",  # 或 glm-4-flash 等
        api_key=api_key,
        base_url="https://open.bigmodel.cn/api/paas/v4/",
    )
    
    resp = llm.invoke("你能帮我解决什么问题")
    print(resp.content)

### Demo-ChromaDB+Embedding-3

    import os
    import time
    
    from langchain_chroma import Chroma
    from langchain_core.prompts import ChatPromptTemplate
    from dotenv import load_dotenv
    from langchain_openai import OpenAIEmbeddings
    from langchain_openai import ChatOpenAI
    from openai import APITimeoutError
    
    load_dotenv()
    api_key = os.getenv("ZHIPUAI_API_KEY")
    if not api_key:
        raise ValueError("请在 .env 或系统环境变量中设置 ZHIPUAI_API_KEY")
    base_url = "https://open.bigmodel.cn/api/paas/v4/"
    # 1. 初始化
    embeddings = OpenAIEmbeddings(
        api_key=api_key,
        model="embedding-3",
        base_url=base_url,
        request_timeout=60,
        max_retries=3,
    )
    vector_store = Chroma(
        collection_name="products",
        embedding_function=embeddings,
        persist_directory="./chroma_db"
    )
    
    # 2. 准备电商知识库
    product_docs = [
        "索尼WH-1000XM5耳机，价格2299元，降噪效果极佳，续航30小时",
        "苹果AirPods Pro 2，价格1899元，支持主动降噪，续航24小时",
        "小米手环8，价格249元，支持心率监测，续航14天",
    ]
    
    # 3. 批量添加（只添加一次，后续可以注释掉）
    vector_store.add_texts(product_docs)
    
    # 4. 检索函数
    def search_products(query: str, k: int = 3) -> list:
        docs = vector_store.similarity_search(query, k=k)
        return [doc.page_content for doc in docs]
    
    # 5. 基于检索结果的问答
    llm = ChatOpenAI(
        model="GLM-4.6V",  # 或 glm-4-flash 等
        api_key=api_key,
        base_url=base_url,
        request_timeout=60,
        max_retries=3,
    )
    
    def ask(query: str) -> str:
        # 检索相关商品
        contexts = search_products(query)
    
        if not contexts:
            return "抱歉，没找到相关信息"
    
        # 构建提示词
        prompt = ChatPromptTemplate.from_template("""
    你是一个电商客服，基于以下商品信息回答问题。
    
    商品信息：
    {context}
    
    用户问题：{query}
    
    要求：
    - 只基于上面的信息回答
    - 如果信息不够，说"我查不到"
    - 价格用¥符号
    """)
    
        prompt_text = prompt.format(context="\n".join(contexts), query=query)
        for attempt in range(3):
            try:
                response = llm.invoke(prompt_text)
                return response.content
            except APITimeoutError:
                if attempt == 2:
                    return "请求超时，请稍后重试"
                time.sleep(2 ** attempt)
    
        return "请求失败，请稍后重试"
    
    # 6. 测试
    print(ask("索尼耳机多少钱？"))
    # 输出：索尼WH-1000XM5耳机的价格是¥2299元。
    
    print(ask("有什么降噪耳机？"))
    # 输出：索尼WH-1000XM5耳机和苹果AirPods Pro 2都支持降噪功能...

### Demo-电商 RAG 问答入门

    agent/
    ├── app/
    │   ├── __init__.py
    │   ├── main.py              # FastAPI 入口
    │   ├── agent.py             # RAG 核心逻辑
    │   ├── vector_store.py      # 向量数据库操作
    │   ├── models.py            # 数据模型（Pydantic）
    │   └── prompts.py           # 提示词模板
    ├── data/
    │   ├── products.json        # 电商数据集
    │   └── init_db.py           # 初始化向量库脚本
    ├── frontend/
    │   └── streamlit_app.py     # Streamlit 界面
    ├── requirements.txt
    └── .env                     # API Key 配置

<!-- 这是一张图片，ocr 内容为： -->

![](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776940927870-1fcae899-3d4f-4aae-ba77-0da1c43d28be.png)

<!-- 这是一张图片，ocr 内容为： -->

![](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776941251157-d392f61e-79c3-4975-90da-28bbfecd5dff.png)

代码可见 [https://github.com/isapeakman/PYProject/tree/fa8d53e837a4ed73d2ae013283e31e2686fc936a](https://github.com/isapeakman/PYProject/tree/fa8d53e837a4ed73d2ae013283e31e2686fc936a)

## 向量数据库

## Embedding模型

用户意图判断

1. 规则+LLM混合运用

规则先判断

命中高确定规则（如“查物流”“开发票”“申请退货退款”）

得到明确意图后，进入对应流程

规则不明确时，交给 LLM

LLM 输出结构化结果：intent、confidence、slots（订单号/商品/原因等）

再由你的后端 Router 决定下一步，不是让 LLM 直接调用高风险操作

根据置信度和参数完整性分流

高置信度 + 参数齐全 -> 执行 API

高置信度 + 参数缺失 -> 追问补齐

低置信度 -> 澄清问题或转人工

置信度计算
