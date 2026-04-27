# TU 01 — Build a Custom RAG Agent

🔑 An agentic RAG loop: LLM decides whether to retrieve, retrieved docs are graded for relevance, irrelevant ones trigger a query rewrite, relevant ones flow into a grounded answer.

## Architecture

Five nodes wired through a `StateGraph(MessagesState)`:

1. `generate_query_or_respond` — model with retriever bound as a tool decides to call it or answer directly.
2. `retrieve` — `ToolNode([retriever_tool])` runs the retrieval.
3. `grade_documents` — conditional edge function (not a node) that scores relevance and routes.
4. `rewrite_question` — reformulates the query when retrieval was off-target, then loops back.
5. `generate_answer` — synthesises the final grounded answer.

## Install & Env

```python
pip install -U langgraph "langchain[openai]" langchain-community langchain-text-splitters bs4
```

```python
import getpass
import os

def _set_env(key: str):
    if key not in os.environ:
        os.environ[key] = getpass.getpass(f"{key}:")

_set_env("OPENAI_API_KEY")
```

## Step 1 — Load and Chunk

```python
from langchain_community.document_loaders import WebBaseLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

urls = [
    "https://lilianweng.github.io/posts/2024-11-28-reward-hacking/",
    "https://lilianweng.github.io/posts/2024-07-07-hallucination/",
    "https://lilianweng.github.io/posts/2024-04-12-diffusion-video/",
]

docs = [WebBaseLoader(url).load() for url in urls]
docs_list = [item for sublist in docs for item in sublist]

text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=100, chunk_overlap=50
)
doc_splits = text_splitter.split_documents(docs_list)
```

## Step 2 — Vector Store + Retriever Tool

```python
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings
from langchain.tools import tool

vectorstore = InMemoryVectorStore.from_documents(
    documents=doc_splits, embedding=OpenAIEmbeddings()
)
retriever = vectorstore.as_retriever()

@tool
def retrieve_blog_posts(query: str) -> str:
    """Search and return information about Lilian Weng blog posts."""
    docs = retriever.invoke(query)
    return "\n\n".join([doc.page_content for doc in docs])

retriever_tool = retrieve_blog_posts
```

`InMemoryVectorStore` is fine for the tutorial; swap in any LangChain vector store (Chroma, pgvector, Qdrant, …) by replacing this block.

## Step 3 — Decide: Retrieve or Answer

```python
from langgraph.graph import MessagesState
from langchain.chat_models import init_chat_model

response_model = init_chat_model("gpt-5.4", temperature=0)

def generate_query_or_respond(state: MessagesState):
    """Call the model to generate a response based on the current state. Given
    the question, it will decide to retrieve using the retriever tool, or simply respond to the user.
    """
    response = (
        response_model
        .bind_tools([retriever_tool]).invoke(state["messages"])
    )
    return {"messages": [response]}
```

`bind_tools` lets the model emit a tool call when it decides retrieval helps. `tools_condition` (later) reads that decision off the message.

## Step 4 — Grade Retrieved Docs

The grader uses structured output to force a binary decision and is wired as a **conditional edge** rather than a node — its return value names the next node.

```python
from pydantic import BaseModel, Field
from typing import Literal

GRADE_PROMPT = (
    "You are a grader assessing relevance of a retrieved document to a user question. \n "
    "Here is the retrieved document: \n\n {context} \n\n"
    "Here is the user question: {question} \n"
    "If the document contains keyword(s) or semantic meaning related to the user question, grade it as relevant. \n"
    "Give a binary score 'yes' or 'no' score to indicate whether the document is relevant to the question."
)

class GradeDocuments(BaseModel):
    """Grade documents using a binary score for relevance check."""
    binary_score: str = Field(
        description="Relevance score: 'yes' if relevant, or 'no' if not relevant"
    )

grader_model = init_chat_model("gpt-5.4", temperature=0)

def grade_documents(state: MessagesState) -> Literal["generate_answer", "rewrite_question"]:
    """Determine whether the retrieved documents are relevant to the question."""
    question = state["messages"][0].content
    context = state["messages"][-1].content

    prompt = GRADE_PROMPT.format(question=question, context=context)
    response = (
        grader_model
        .with_structured_output(GradeDocuments).invoke(
            [{"role": "user", "content": prompt}]
        )
    )
    score = response.binary_score

    if score == "yes":
        return "generate_answer"
    else:
        return "rewrite_question"
```

## Step 5 — Rewrite the Question

```python
from langchain.messages import HumanMessage

REWRITE_PROMPT = (
    "Look at the input and try to reason about the underlying semantic intent / meaning.\n"
    "Here is the initial question:"
    "\n ------- \n"
    "{question}"
    "\n ------- \n"
    "Formulate an improved question:"
)

def rewrite_question(state: MessagesState):
    """Rewrite the original user question."""
    messages = state["messages"]
    question = messages[0].content
    prompt = REWRITE_PROMPT.format(question=question)
    response = response_model.invoke([{"role": "user", "content": prompt}])
    return {"messages": [HumanMessage(content=response.content)]}
```

The rewrite is appended as a new `HumanMessage`, so the next pass through `generate_query_or_respond` sees it as the latest user turn.

## Step 6 — Generate the Final Answer

```python
GENERATE_PROMPT = (
    "You are an assistant for question-answering tasks. "
    "Use the following pieces of retrieved context to answer the question. "
    "If you don't know the answer, just say that you don't know. "
    "Use three sentences maximum and keep the answer concise.\n"
    "Question: {question} \n"
    "Context: {context}"
)

def generate_answer(state: MessagesState):
    """Generate an answer."""
    question = state["messages"][0].content
    context = state["messages"][-1].content
    prompt = GENERATE_PROMPT.format(question=question, context=context)
    response = response_model.invoke([{"role": "user", "content": prompt}])
    return {"messages": [response]}
```

## Step 7 — Wire the Graph

```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode, tools_condition

workflow = StateGraph(MessagesState)

workflow.add_node(generate_query_or_respond)
workflow.add_node("retrieve", ToolNode([retriever_tool]))
workflow.add_node(rewrite_question)
workflow.add_node(generate_answer)

workflow.add_edge(START, "generate_query_or_respond")

workflow.add_conditional_edges(
    "generate_query_or_respond",
    tools_condition,
    {
        "tools": "retrieve",
        END: END,
    },
)

workflow.add_conditional_edges(
    "retrieve",
    grade_documents,
)

workflow.add_edge("generate_answer", END)
workflow.add_edge("rewrite_question", "generate_query_or_respond")

graph = workflow.compile()
```

The two conditional edges are the agentic part:

- `tools_condition` reads the latest assistant message — if it has tool calls, route to `retrieve`, otherwise end.
- `grade_documents` returns `"generate_answer"` or `"rewrite_question"` directly, so no destination map is needed.

`rewrite_question -> generate_query_or_respond` closes the loop, so a bad retrieval triggers a fresh query attempt.

## Step 8 — Run It

```python
for chunk in graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "What does Lilian Weng say about types of reward hacking?",
            }
        ]
    }
):
    for node, update in chunk.items():
        print("Update from node", node)
        update["messages"][-1].pretty_print()
        print("\n\n")
```

Switch to `stream_mode="messages"` for token-level output (see [[LangGraph PT 01 - Streaming]]).

## Patterns Worth Stealing

- **`MessagesState`** as the conversation log — every node appends, nothing is mutated.
- **Conditional edges over branching nodes** — `tools_condition` and `grade_documents` keep routing logic out of node bodies.
- **`with_structured_output(GradeDocuments)`** — Pydantic v2 schema turns the grader into a deterministic classifier.
- **Tool binding** — `response_model.bind_tools([retriever_tool])` is the only thing that makes the model "agentic" here.
- **Loop on failure** — `rewrite_question` flowing back into `generate_query_or_respond` is the difference between RAG and *agentic* RAG.

💡 Agentic RAG = retrieval as a tool + a relevance grader on the retrieved context + a rewrite-and-retry edge. The graph stays tiny; the intelligence lives in the conditional edges.
