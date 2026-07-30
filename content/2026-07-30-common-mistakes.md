Title: Common LangChain Beginner Mistakes (and How to Avoid Them)
Date: 2026-07-30
Category: GenAI
Tags: GenAI, LLM, LangChain, Debugging, Best Practices, Python, AI Engineering
Slug: common-langchain-beginner-mistakes
Status: draft

LangChain has a shallow learning curve at the start and a steep one shortly after. The first chain works quickly. Then something breaks and the error message points nowhere useful. Most of those moments trace back to the same handful of mistakes.

## Mixing up embed_query and embed_documents

Both methods exist on every LangChain embedder. Beginners often use `embed_query` for everything — including indexing. It works, but some models apply different preprocessing to each method. `embed_documents` is for batching your corpus at index time. `embed_query` is for single inputs at retrieval time. Use them as intended.

```python
# Wrong — using embed_query for a batch of documents
vectors = [embedder.embed_query(doc) for doc in documents]

# Right
vectors = embedder.embed_documents(documents)
```

## Switching embedding models mid-project

Stored vectors are tied to the model that generated them. If you index your documents with `text-embedding-3-small` and later switch to `all-MiniLM-L6-v2`, the vectors from the old model and the query vectors from the new one live in different spaces. Similarity search returns garbage — and it fails silently.

Pick an embedding model early, write it down somewhere, and re-embed from scratch if you change it. There's no shortcut.

## Ignoring chunk boundaries in RAG

The most common RAG failure pattern: the answer is clearly in the document, the model still misses it. Nine times out of ten, the answer is split across two chunks and only one was retrieved.

Set `chunk_overlap` to at least 10–20% of your `chunk_size`. Then actually look at what your retriever returns — print the chunks before you wire them into the LLM. If the retrieved text doesn't contain the answer, the problem is the retrieval layer, and tuning the prompt won't fix it.

```python
# Quick way to inspect retrieved chunks
results = vectorstore.similarity_search("your query here", k=3)
for r in results:
    print(r.page_content)
    print("---")
```

## Not setting output_key in SequentialChain

`SequentialChain` passes outputs between steps using named keys. If you forget to set `output_key` on an intermediate chain, LangChain uses a default that may conflict with another step's input variable. The chain runs, produces wrong output, and doesn't tell you why.

Every `LLMChain` inside a `SequentialChain` needs an explicit `output_key`. Name them clearly and make sure they match the `input_variables` of the next chain in line.

```python
# Missing output_key — will cause silent errors downstream
name_chain = LLMChain(llm=llm, prompt=name_prompt)

# Correct
name_chain = LLMChain(llm=llm, prompt=name_prompt, output_key="product_name")
```

## Using ConversationBufferMemory for long sessions

`ConversationBufferMemory` keeps the full transcript. For a 5-turn demo, that's fine. For a 50-turn session, you're sending thousands of tokens of history on every call — most of it irrelevant. Eventually you hit the context window limit and the chain fails.

If your sessions can run long, use `ConversationSummaryBufferMemory` from the start. It keeps recent messages verbatim and compresses older ones into a summary. The token cost stays bounded without losing all historical context.

## Forgetting temperature=0 for structured outputs

When you need the model to return JSON, a list, or any other structured format, set `temperature=0`. Higher temperatures introduce randomness that breaks parsers — you get valid JSON nine times out of ten and malformed output on the tenth, which crashes your pipeline at an unpredictable moment.

```python
llm = OpenAI(temperature=0)  # deterministic output for structured tasks
```

For creative tasks, turn temperature up. For anything that gets parsed downstream, keep it at zero.

## Treating LangChain errors as LLM errors

When a chain throws, the error is often somewhere in the scaffolding — a missing variable in a prompt template, a key mismatch in a sequential chain, a retriever returning an unexpected type. Beginners often blame the model and start tweaking the prompt.

Enable verbose mode first:

```python
chain = ConversationChain(llm=llm, memory=memory, verbose=True)
```

`verbose=True` prints every step — the rendered prompt, the raw model output, intermediate values. Most errors become obvious once you see what's actually being sent. If verbose isn't enough, LangSmith gives you the full trace with token counts and latencies.

## Hardcoding API keys

It happens in tutorials, then it stays in production code. Set keys as environment variables — never in the source file.

```python
# Wrong
llm = OpenAI(api_key="sk-...")

# Right
import os
llm = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
```

Use a `.env` file with `python-dotenv` during development. If a key ends up in a git commit, rotate it immediately — GitHub scans for leaked keys and so do a lot of other parties.

## Building the full pipeline before testing components

A RAG chain has five or six moving parts. Beginners often assemble all of them before running anything. When it fails, there's no obvious place to start.

Test each component in isolation first. Confirm the splitter produces sensible chunks. Confirm the embedder returns vectors. Confirm the retriever finds relevant results for a known query before you wire it into a chain. By the time you assemble the full pipeline, you know every piece works independently.

## Not reading the LCEL migration docs

LangChain Expression Language (LCEL) is the current way to compose chains — using the pipe operator (`|`) instead of class-based chains like `LLMChain` and `SequentialChain`. A lot of tutorials online still use the old API because they haven't been updated.

If you're starting a new project now, use LCEL:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Explain {topic} in two sentences.")
llm = ChatOpenAI(temperature=0)
parser = StrOutputParser()

chain = prompt | llm | parser
result = chain.invoke({"topic": "gradient descent"})
```

The class-based API still works, but LCEL is what LangChain is actively developing. Learn the current thing, not the thing that tutorials from 18 months ago showed.