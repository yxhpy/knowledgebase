# 构造自己的LLM供给LangChain使用

## 你可能需要预备的知识

* `Python3`基本语法
* `LangChain`了解 [官方文档](https://python.langchain.com/docs/get_started/introduction.html)
    * 文档加载器
    * 分片器
    * `embedding`
    * `llm`
* 面向对象 `封装`、`继承`、`多态`
* [安装`LangChain`使用`ZhipuAi`去实现一个知识库](zh-cn/大模型/LangChain/利用ZhipuAi完成知识库建立.md) 
* [对比一下官网如何建立知识库](https://python.langchain.com/docs/use_cases/question_answering/) 

## 为什么要自己去实现一些类、方法，`LangChain`现有不能满足我们？
1. `LangChain`对于`OpenAi`支持是最好的，对其它大模型支持非常有限，尤其是国产。
2. `OpenAi`国内无法使用，并且要使用`Api`需要付比较昂贵的费用，对于学习不利。
3. 助力国产对接希望能与国际接轨，能将大模型尽快应用于生产之中。
## 为什么要学习`LangChain`
1. 和AI对话我们经常发现AI的回答是不可控的，相同的问题AI能给出不同答案，对于我们来程序员来说这样不可控的答案是不能用于程序开发的，因此需要将提示词放到对话中从而修正AI输出。
2. AI对于大文本的接受能力是有限的，因此需要我们通过各种压缩、分割等方式让AI更容易提取信息。
3. 流式处理，极简化实现AI程序开发
* 下列例子完成了，提示模板生成、对接AI、输出解析
```java
chain = (
        {"doc": lambda x: x.page_content} 
        | ChatPromptTemplate.from_template(
            template="""按照文档内容，帮我生成3到10个具有全面、精练、专业的问题。\n\n{doc}\n\n{instructions}""",
            partial_variables={"instructions": json_string_list.get_format_instructions()})
        | ZhipuAi()
        | JsonStringListOutParser()
)
```

## 不浪费大家时间大家可以拿来直接使用
* 后续出章节从源码角度给大家为什么要这么写
### 创建一个ChatModel

```python
class ZhipuAi(BaseChatModel, ABC):

    def _generate(self, messages: List[BaseMessage], stop: Optional[List[str]] = None,
                  run_manager: Optional[CallbackManagerForLLMRun] = None, **kwargs: Any) -> ChatResult:
        generations = []
        for i in messages:
            chat_msg = SystemMessage(content=get_question_answer({"role": "system", "content": i.content}))
            generations.append(ChatGeneration(message=chat_msg))
        return ChatResult(generations=generations)

    @property
    def _llm_type(self) -> str:
        return "zhipu"
```

### 创建一个LLM

```python
class ZhipuLLM(LLM, ABC):

    def _call(self, prompt: str, stop: Optional[List[str]] = None,
              run_manager: Optional[CallbackManagerForLLMRun] = None, **kwargs: Any) -> str:
        return get_question_answer({"role": "user", "content": prompt})

    @property
    def _llm_type(self) -> str:
        return "zhipu"
```

### 对接智谱AI接口

```python
def get_question_answer(prompt):
    zhipuai.api_key = ""
    response = zhipuai.model_api.sse_invoke(
        model=chatglm_pro,
        prompt=prompt
    )
    res = ""
    for event in response.events():
        if event.event == "add":
            print(event.data, end="")
            res += event.data
        elif event.event == "error" or event.event == "interrupted":
            print(event.data, end="")
        elif event.event == "finish":
            print(event.data, end="\n")
            # print(event.meta)
            res += event.data
        else:
            print(event.data, end="")
    return res
```