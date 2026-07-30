Title: Embeddings Explained for LangChain Beginners
Date: 2026-07-30
Category: GenAI
Tags: GenAI, LLM, LangChain, Embeddings, RAG, Vector Search, Python, AI Engineering
Slug: embeddings-explained-langchain-beginners


Before you can build a retrieval system, you need to understand what embeddings are and why they exist. Skip this and the rest of RAG won't make sense — you'll be configuring things without knowing what they do.

## What an embedding is

An embedding is a list of numbers that represents the meaning of a piece of text. A sentence, a paragraph, a document — each one gets converted into a fixed-length vector of floats, typically hundreds or thousands of dimensions long.

The useful property: texts that mean similar things end up with vectors that are close together in that high-dimensional space. "The cat sat on the mat" and "A feline rested on the rug" will have similar embeddings. "Stock market futures" will be far away from both.

This is what makes semantic search possible. You're not matching keywords — you're measuring meaning distance.

## Why LangChain needs them

LangChain's retrieval pipeline works like this:

1. Split documents into chunks
2. Embed each chunk into a vector
3. Store those vectors in a vector database
4. At query time, embed the user's question
5. Find the chunks whose vectors are closest to the question vector
6. Pass those chunks to the LLM as context

Steps 2 and 4 are where embeddings come in. The same embedding model has to handle both — if you embed your documents with one model and your queries with another, the vectors live in different spaces and the similarity search breaks.

## Using embeddings in LangChain

LangChain wraps embedding models behind a common interface. Here's the OpenAI embedder:

```python
from langchain_openai import OpenAIEmbeddings

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

# Embed a single query
query_vector = embedder.embed_query("What is gradient descent?")

# Embed a list of documents
doc_vectors = embedder.embed_documents([
    "Gradient descent is an optimization algorithm.",
    "Neural networks learn by adjusting weights.",
    "Python is a general-purpose programming language."
])

print(len(query_vector))       # 1536 for text-embedding-3-small
print(len(doc_vectors))        # 3 vectors
print(len(doc_vectors[0]))     # 1536 floats per vector
```

`embed_query` is for single inputs at retrieval time. `embed_documents` is for batching your corpus during indexing. The distinction is minor — some models apply different preprocessing to each — but it's the correct way to call the interface.

## Using a local embedding model

You don't need an API for embeddings. HuggingFace models run locally and are free.

```python
from langchain_huggingface import HuggingFaceEmbeddings

embedder = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")

vector = embedder.embed_query("How do transformers work?")
print(len(vector))  # 384
```

`all-MiniLM-L6-v2` is a solid starting point — small, fast, and good enough for most tasks. The vectors are 384-dimensional instead of 1536, which means smaller storage and faster search at the cost of some precision.

For production use cases where accuracy matters more than latency or cost, `text-embedding-3-large` from OpenAI or `bge-large-en-v1.5` from BAAI are worth testing.

## What similarity actually means

Once you have vectors, you need a way to measure how close two of them are. The two most common measures:

**Cosine similarity** — measures the angle between vectors. Two vectors pointing in the same direction score 1.0, regardless of their magnitude. This is the default for most embedding models because it handles texts of different lengths gracefully.

**Dot product** — measures both direction and magnitude. Faster to compute, but sensitive to vector scale. Some models (like OpenAI's) are trained to use dot product — check the model docs before assuming cosine is correct.

In practice, LangChain and most vector databases handle this for you. But knowing which one your retriever uses matters when you're debugging why certain results rank unexpectedly.

## Embeddings are frozen at index time

One thing that trips people up: the embedding model doesn't run at query time on your stored documents. You embed documents once, store the vectors, and those vectors stay fixed. The model only runs again when you embed a new query or add new documents.

This means if you switch embedding models after indexing, you have to re-embed your entire document store. The old vectors and new query vectors are incompatible — they live in different spaces. Build your pipeline with a specific model chosen early, or you'll be re-indexing a lot.

## A complete example

Here's the full flow from documents to query — embeddings included:

```python
from langchain_openai import OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Your raw text
documents = [
    "LangChain is a framework for building LLM applications.",
    "FAISS is a library for efficient vector similarity search.",
    "Embeddings convert text into numerical vectors.",
    "RAG combines retrieval with language model generation."
]

# Split (skip for short texts, but good practice)
splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=20)
chunks = splitter.create_documents(documents)

# Embed and store
embedder = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(chunks, embedder)

# Query
query = "What is RAG?"
results = vectorstore.similarity_search(query, k=2)

for r in results:
    print(r.page_content)
```

`FAISS.from_documents` calls `embed_documents` under the hood and builds the index. `similarity_search` calls `embed_query` on your question, then finds the closest vectors. The LLM step would come after — you pass `results` as context into a prompt.

## Choosing an embedding model

A few things to weigh:

- **Dimension size** — higher dimensions capture more nuance but cost more storage and compute. 384 is fine for prototypes; 1536 or 3072 for production.
- **Max input tokens** — most models cap at 512 or 8192 tokens per chunk. Chunks longer than the cap get silently truncated. Know your model's limit before indexing.
- **Domain fit** — general models handle general text well. For legal, medical, or code-heavy corpora, look for domain-specific models or test a few before committing.
- **Cost** — local models are free to run. API-based models charge per token. At scale, that adds up.

The embedding layer looks like configuration. It's actually one of the most load-bearing decisions in a RAG pipeline.