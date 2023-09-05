# 利用`ZhipuAi`完成知识库建立

## 你可能需要参考
* [ZhipuAi官方文档](https://open.bigmodel.cn/)
* [官网如何建立知识库](https://python.langchain.com/docs/use_cases/question_answering/)
* 有任何问题，麻烦大家给出建议
## 创建加载文件

* 这里将文件夹里面的`.docx`文件加载出来，`use_multithreading`使用多线程，`show_progress`展示加载进度，`List[Document]`返回格式。

```python
class Document(Serializable):
    """Class for storing a piece of text and associated metadata."""

    page_content: str
    """String text."""
    metadata: dict = Field(default_factory=dict)
    """Arbitrary metadata about the page content (e.g., source, relationships to other
        documents, etc.).
    """
```

```python
def load_doc() -> List[Document]:
    loader = DirectoryLoader(path=r'G:\work\ai\zhipu_01\threejs', glob="**/*.docx", use_multithreading=True,
                             show_progress=True)
    return loader.load()
```

## 分割文本

* 将大段文本分割为小段文本，小段文本对于AI来说更容易理解，长文本可能会导致AI总结得不是很好。
* 下面的例子是使用`Markdown`文本分割器，`chunk_size`文段大小被分割成了2000，`chunk_overlap`交集分割内容。

```python
def doc_split(data: List[Document]) -> List[Document]:
    chunk_size = 2000
    text_splitter = MarkdownTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_size * 0.2,
        length_function=len,
        add_start_index=True)
    return text_splitter.split_documents(data)
```

## 创建embedding

* 创建`Embeddings`这里我使用的是`m3e-base`

```python
def get_embedding() -> Embeddings:
    global embedding
    model_name = r"G:\work\Langchain-Chatchat\model\m3e-base"
    model_kwargs = {'device': 'cuda'}
    encode_kwargs = {'normalize_embeddings': False}
    if embedding is None:
        embedding = HuggingFaceEmbeddings(
            model_name=model_name,
            model_kwargs=model_kwargs,
            encode_kwargs=encode_kwargs
        )
    return embedding
```

* 最简单的方式可以使用`OpenAi`的`Embedding`

```python
def get_embedding() -> Embeddings:
    return OpenAIEmbeddings()
```

## 创建向量存储

* `FAISS`（Facebook AI Similarity Search）是Facebook AI团队开源的针对聚类和相似性搜索库。
* `FAISS` 针对高维空间中的海量数据，提供了高效且可靠的检索方法，为稠密向量提供高效相似度搜索和聚类，支持十亿级别向量的搜索，是目前较成熟的近似近邻搜索库。它包含多种搜索任意大小向量集的算法，以及用于算法评估和参数调整的支持代码。
* `FAISS`用C++编写，并提供与Numpy完美衔接的Python接口。除此以外，对一些核心算法提供了GPU实现

```python
def to_vectorstore(
        documents: List[Document]
) -> VectorStore:
    return FAISS.from_documents(
        documents=documents,
        embedding=get_embedding()
    )
```

## 从向量存储中尝试找出与用户输入的`prompt`近似的文段

* 用户输入问题后，返回5条近似的文段

```python
docs = vectorstore.similarity_search(question, k=5)
```

## 将文段与用户输入合并加入到更大的`prompt`中

* `content`为文段内容
* `question`用户问题

```python
def get_template() -> PromptTemplate:
    template = PromptTemplate(
        template="""请仔细阅读我给予的文档，我将提出一些问题，请按照文档内容进行回答，如果文档内容无法回答我的问题请直接回答“我不知道”，不允许按照自己胡编乱造："
                 "{content}"
                 "我的问题是：”{question}“"""
                 "你的回答：""",
        input_variables=["question", "content"]
    )
    return template
```

* 格式化`PromptTemplate`为文本

```python
def ask_a_question(question: str, docs: list[str]):
    content = '\n'.join([f"{idx + 1}、{i}" for idx, i in enumerate(docs)])
    content = get_template().format(question=question, content=content)
    return llm(content)
```

## 将整合的`prompt`给`ZhipuLLM`

* 这里需要继承一下`LLM`并且实现`_call`与`_llm_type`

```python
class ZhipuLLM(LLM, ABC):

    def _call(self, prompt: str, stop: Optional[List[str]] = None,
              run_manager: Optional[CallbackManagerForLLMRun] = None, **kwargs: Any) -> str:
        return get_question_answer({"role": "user", "content": prompt})

    @property
    def _llm_type(self) -> str:
        return "zhipu"


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

## 总结
* 虽然我们可以使用上列代码完成一个简单的知识库，但返回的答案依然不是很准确，我们需要利用提示词工程、压缩、微调等方式来让AI更好的去管理我们的知识库。后续我们慢慢讲解