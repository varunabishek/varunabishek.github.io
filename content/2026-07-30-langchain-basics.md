Title: LangSmith Basics: Debugging and Tracing Your Chains
Date: 2026-07-30
Category: GenAI
Tags: GenAI, LLM, LangChain, LangSmith, Debugging, Tracing, Observability, Python, AI Engineering
Slug: langsmith-basics-debugging-tracing-chains


LLM applications fail in ways that are hard to see. A chain returns a bad answer and you don't know whether the prompt was wrong, the retriever missed, or the model just hallucinated. LangSmith is how you look inside.

## What LangSmith is

LangSmith is Anthropic's observability platform for LangChain — it records every step of your chain as a trace. Prompt inputs, LLM outputs, retrieval results, tool calls, token counts, latency — all of it, in one place, for every run.

You don't change how your chain is written. You add a few environment variables and tracing happens automatically.

## Setting it up

Create an account at [smith.langchain.com](https://smith.langchain.com) and grab an API key from the settings page. Then set these before your chain runs:

```python
import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_ENDPOINT"] = "https://api.smith.langchain.com"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key-here"
os.environ["LANGCHAIN_PROJECT"] = "my-first-project"
```

`LANGCHAIN_PROJECT` is just a label — traces get grouped under it in the UI. Use something descriptive: the chain name, the feature, or the experiment you're running.

After that, run any LangChain code. The traces show up in your project dashboard automatically.

## What a trace looks like

Every chain run produces a tree of spans. The root span is the chain itself. Under it, you'll see child spans for each step — LLM calls, retrieval operations, tool invocations, memory reads.

Each span shows:

- **Input** — what was passed in at that step
- **Output** — what came back
- **Latency** — how long it took
- **Token usage** — prompt tokens, completion tokens, total
- **Errors** — if the step threw, the full traceback

For a simple `ConversationChain`, the trace is shallow — one LLM call with the formatted prompt and the raw response. For a multi-step agent, the tree can go several levels deep with tool calls branching off the main path.

## Debugging a bad output

Say your RAG chain returns a wrong answer. Without tracing, you're guessing. With LangSmith, you open the trace and work backwards:

1. Look at the LLM span — what prompt did the model actually receive? Was the context there?
2. Look at the retrieval span — what chunks came back? Were they relevant?
3. Look at the query — did the question get reformulated somewhere upstream?

Most of the time, the problem is obvious once you see the actual prompt. A template bug, a missing variable, a retriever that returned garbage — these all show up immediately.

```python
from langchain.chains import RetrievalQA
from langchain_openai import OpenAI, OpenAIEmbeddings
from langchain.vectorstores import FAISS

llm = OpenAI(temperature=0)
embedder = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(
    ["LangSmith traces every step of your LangChain runs."],
    embedder
)
retriever = vectorstore.as_retriever()

chain = RetrievalQA.from_chain_type(llm=llm, retriever=retriever)

# This run will appear in LangSmith with full retrieval + LLM trace
result = chain.run("What does LangSmith do?")
print(result)
```

Open the project in the UI after this runs. You'll see the retrieval span with the exact chunks that were fetched, then the LLM span with the fully-rendered prompt those chunks were injected into.

## Adding metadata and tags

Traces are more useful when you can filter them. LangSmith lets you attach metadata and tags to runs:

```python
from langchain.callbacks.manager import collect_runs

with collect_runs() as cb:
    result = chain.run(
        "What does LangSmith do?",
        tags=["production", "rag-v2"],
        metadata={"user_id": "u_123", "session_id": "s_456"}
    )
    run_id = cb.traced_runs[0].id
    print(f"Trace ID: {run_id}")
```

Tags and metadata let you slice your traces in the UI — filter by environment, by user, by chain version. When you're running experiments, tags are how you keep the results organized.

## Comparing runs

LangSmith has a comparison view. Select two runs from the same project and it shows you inputs, outputs, and latencies side by side. This is useful when you change a prompt and want to know whether the new version actually improved things — not just on the one example you tested manually, but across a batch.

## Datasets and evaluation

Once you have traces, you can promote individual runs to a dataset. Pick a run where the output was correct, save it as an example, and build up a test set over time. LangSmith can then run your chain against the dataset and score outputs automatically.

```python
from langsmith import Client

client = Client()

# Create a dataset
dataset = client.create_dataset("rag-eval-set")

# Add an example
client.create_example(
    inputs={"query": "What does LangSmith do?"},
    outputs={"answer": "LangSmith traces every step of LangChain runs."},
    dataset_id=dataset.id
)
```

From here you can run evaluations programmatically or trigger them from the UI. It's a short path from "this answer looked right" to a repeatable regression test.

## What to check first when things break

In rough order of how often each one is the actual problem:

- **Formatted prompt** — look at the LLM input span. Is the prompt what you expected? Template bugs show up here.
- **Retrieved chunks** — if the answer is in your documents but the model missed it, check what the retriever actually returned.
- **Token counts** — if a run is slow or expensive, check whether a prompt is bloated. Memory accumulation and large context injections show up clearly here.
- **Latency by step** — if a chain is slow, the latency breakdown shows exactly which step is the bottleneck.

LangSmith won't fix your chain for you, but it removes the guesswork. Most debugging sessions that used to take an hour of print statements take five minutes once you can see every step.