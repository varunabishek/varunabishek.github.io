Title: Text Splitters Explained: Why Chunking Strategy Matters
Date: 2026-07-30
Category: GenAI
Tags: GenAI, LLM, LangChain, RAG, Text Splitters, Chunking, Python, AI Engineering
Slug: text-splitters-chunking-strategy


When you build a retrieval-augmented system, most of the tuning work happens before the LLM ever sees a document. How you split text determines what gets retrieved — and bad splits mean bad answers, even with a great model.

## The problem with long documents

LLMs have a context window. You can't dump a 50-page PDF into a prompt. So you split it into chunks, embed each chunk, store them in a vector database, and retrieve the most relevant ones at query time.

The question is: how do you split? Cut in the wrong place and you get chunks that are missing context, start mid-sentence, or bury the answer across a boundary where no single retrieval will find it. The model can only work with what you hand it.

## CharacterTextSplitter

The most basic option. It splits on a character — usually a newline or a space — and tries to keep chunks under a specified size.

```python
from langchain.text_splitter import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n",
    chunk_size=500,
    chunk_overlap=50
)

chunks = splitter.split_text(your_text)
```

`chunk_size` controls how big each chunk gets (in characters). `chunk_overlap` is how much text repeats between adjacent chunks — this matters because answers sometimes straddle a boundary, and overlap gives the retriever a chance to catch them.

It works. It's also blunt. A `\n` split doesn't know whether it's cutting between paragraphs or in the middle of a table.

## RecursiveCharacterTextSplitter

This is the one LangChain recommends by default — and for good reason. Instead of one separator, it tries a list of them in order: `["\n\n", "\n", " ", ""]`. It starts with paragraph breaks, falls back to line breaks, then words, then characters.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50
)

chunks = splitter.split_text(your_text)
```

The result is chunks that respect natural text structure wherever possible. A paragraph that fits under the size limit stays together. Only when it's too big does the splitter go finer. That behavior makes retrieved chunks far more coherent than character-level splits.

Use this as your starting point for most plain-text documents.

## TokenTextSplitter

`CharacterTextSplitter` measures chunk size in characters. But LLMs charge for tokens, and token count doesn't map cleanly to character count — especially across languages, code, or punctuation-heavy text.

`TokenTextSplitter` measures in tokens instead.

```python
from langchain.text_splitter import TokenTextSplitter

splitter = TokenTextSplitter(chunk_size=200, chunk_overlap=20)
chunks = splitter.split_text(your_text)
```

When your context window budget is tight and you're working with text that has variable token density — technical docs, mixed-language content, code — this gives you more precise control.

## MarkdownHeaderTextSplitter

Structure-aware splitting. For Markdown documents, this one splits on headers (`#`, `##`, `###`) and carries the header metadata into each chunk.

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]

splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
chunks = splitter.split_text(your_markdown)
```

Each chunk knows which section it came from. When you store these in a vector database, you can filter by section at retrieval time or include the header in the chunk text so the LLM has context about where the content lives.

Good fit for technical documentation, wikis, or any structured Markdown corpus.

## PythonCodeTextSplitter / Language splitters

Code has its own structure — functions, classes, blocks. Splitting it on character count will cut through function definitions and break the semantic unit the retriever should be working with.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter, Language

splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=500,
    chunk_overlap=50
)

chunks = splitter.split_text(your_python_code)
```

LangChain supports multiple languages here — Python, JS, Markdown, HTML, and others. The splitter knows the syntax boundaries for each and tries to keep logical units intact.

## Why chunk_overlap matters

Every splitter has a `chunk_overlap` parameter and it's worth taking seriously. If your chunks are 500 characters with zero overlap, a sentence that starts at character 495 of one chunk and ends at character 10 of the next is effectively unfindable — no single retrieved chunk contains it whole.

Overlap duplicates some text across adjacent chunks so that boundary content shows up fully in at least one of them. A 10–20% overlap is a reasonable default. Bigger overlaps help retrieval quality but increase your vector store size and embedding cost.

## The actual impact on retrieval quality

Chunking strategy doesn't change what the LLM knows — it changes what the retriever can find. A model can only answer from what gets passed to it. If the relevant passage is split across two chunks and only one is retrieved, you get a partial answer or no answer. If chunks are too large, retrieved content is noisy and the signal gets buried.

Most RAG failures that look like model failures are actually retrieval failures. And most retrieval failures trace back to chunking. Get this layer right before you start tuning embeddings or prompts.

A reasonable starting point: `RecursiveCharacterTextSplitter` with `chunk_size=500`, `chunk_overlap=50`, then look at what your actual retrieved chunks contain. If the model keeps missing things that are clearly in the document, the chunk boundary is usually why.