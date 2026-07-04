Title: Stop Feeding Your LLM Guesses. Feed It Facts.
Date: 2026-07-04
Category: GenAI
Tags: GenAI, AI-agents, LLM, agentic-systems, design-patterns, reliability
Slug: Stop-Feeding-Your-LLM-Guesses.-Feed-It-Facts.


# The trick that turns a language model into a research assistant.


Large language models are great at reasoning and writing, but they only know what they were trained on. Ask one about a document it's never seen, or something that happened after its training cutoff, and it will either say it doesn't know or, worse, make something up.

Retrieval-Augmented Generation fixes this. Instead of relying only on the model's memory, you hand it relevant information at the moment you ask the question. The model reads that information and answers based on it. That's the whole idea. Everything else is implementation detail.

This article walks through building a simple RAG app: one that answers questions about a PDF you give it.

# What RAG actually does:

A RAG system has two jobs: find the right information, then generate an answer using it.

Say you upload a 50-page company handbook and ask "how many vacation days do new hires get?" The system doesn't feed the entire handbook to the model — that would be slow and expensive, and most of it is irrelevant to the question. Instead, it searches the handbook for the few paragraphs most likely to contain the answer, and only sends those to the model along with your question.

The model then reads that small chunk of text and answers, grounded in what it just read rather than guessing from memory.

# The four steps:

Every RAG pipeline, no matter how complex it eventually gets, boils down to four steps:

**1. Load the document**. Pull in your source material — a PDF, a website, a spreadsheet, whatever you're working with.

**2. Split it into chunks.** Documents are usually too long to search effectively as one block. You break them into smaller pieces, often a few hundred words each, sometimes overlapping slightly so you don't cut a sentence in half.

**3. Embed and store the chunks.** Each chunk gets converted into a vector — a list of numbers that captures its meaning. These vectors go into a vector database. This is what lets you search by meaning instead of exact keyword matches.

**4. Retrieve and generate.** When a question comes in, you convert it into a vector too, find the stored chunks whose vectors are closest to it, and pass those chunks plus the question to the language model. The model generates an answer using that context.

## Building it: 

Here's what this looks like in code, using LangChain, OpenAI's embeddings, and a local vector store called Chroma.

## Step 1: Install what you need

**bashpip install langchain langchain-openai langchain-community chromadb pypdf**

## Step 2: Load your document

**pythonfrom langchain_community.document_loaders import PyPDFLoader**

loader = PyPDFLoader("handbook.pdf")
documents = loader.load()

documents is now a list of page objects, each holding the text from one page of the PDF.

## Step 3: Split into chunks

**pythonfrom langchain.text_splitter import RecursiveCharacterTextSplitter**

**splitter = RecursiveCharacterTextSplitter(**
    **chunk_size=500,**
    **chunk_overlap=50**
)
**chunks = splitter.split_documents(documents)**

The overlap matters more than it looks. Without it, a sentence that spans two chunks can get cut in a way that loses meaning in both pieces.

## Step 4: Embed and store

**pythonfrom langchain_openai import OpenAIEmbeddings**
**from langchain_community.vectorstores import Chroma**

**embeddings = OpenAIEmbeddings()**
**vectorstore = Chroma.from_documents(chunks, embeddings)**

This one block does the heavy lifting: it turns every chunk into a vector and saves it so you can search it later.

## Step 5: Set up retrieval and generation

**pythonfrom langchain_openai import ChatOpenAI**
**from langchain.chains import RetrievalQA**

**llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)**

**qa_chain = RetrievalQA.from_chain_type(**
    **llm=llm,****
    **retriever=vectorstore.as_retriever(),**
    **return_source_documents=True**
**)**

## Step 6: Ask a question

**pythonresult = qa_chain.invoke({"query": "How many vacation days do new hires get?"})**
**print(result["result"])**

That's a working RAG app. Feed it a PDF, ask it questions, and it answers based on the actual document rather than a guess.

## Why it works better than just asking the model directly:

Without retrieval, the model has no way to know what's in your specific PDF. It might recognize general patterns about vacation policies, but it can't tell you your company's actual number.

With retrieval, the model sees the relevant paragraph before answering. It's the difference between asking someone to recall a policy from memory versus handing them the policy document and asking them to read the relevant page first.

This also cuts down on hallucination. A model working from a document in front of it is far less likely to invent details than one working purely from its training data.

## A few things to watch for:

Chunk size matters more than you'd think. Too small, and chunks lose context — a number without the sentence explaining what it refers to. Too large, and you dilute the search: the model retrieves a big chunk where only one sentence is actually relevant, and the rest is noise.

Retrieval isn't perfect. If your question uses different words than the document, the vector search might miss the right chunk. Testing with a range of phrasings helps you catch this early.

More chunks isn't always better. Retrieving the top 10 chunks instead of the top 3 sounds safer, but it can bury the model in irrelevant text and make the answer worse, not better.

## Where to go from here:

Once this basic version works, a few natural upgrades:


Swap in a hosted vector database like Pinecone or Weaviate if you need to scale past a local file.

Add a step that shows which source chunks the answer came from, so users can verify it.

Try different embedding models — some handle domain-specific language (legal, medical, technical) better than general-purpose ones.

Experiment with re-ranking retrieved chunks before sending them to the model, which can noticeably improve answer quality on tricky questions.


None of these are required to get started. The six steps above are enough to go from a static PDF to a system you can actually ask questions and get real answers from.

