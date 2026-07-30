Title: Giving Your Chatbot Memory: ConversationBufferMemory vs. Other Memory Types
Date: 2026-07-30
Category: GenAI
Tags: GenAI, LLM, LangChain, Memory, Chatbot, Python, AI Engineering
Slug: chatbot-memory-conversationbuffermemory-vs-other-types

By default, an LLM has no idea what you said two messages ago. Every call is stateless — the model processes whatever is in the current prompt and nothing else. If you want a chatbot that remembers the conversation, you have to build that yourself. LangChain's memory modules are how you do it.

## Why memory is a design decision

There's no single right way to handle memory. A short customer support chat needs something different from a long-running research assistant. The tradeoff is almost always the same: how much context do you keep vs. how much do you spend on tokens?

LangChain gives you several memory types. Here's what each one does and when to reach for it.

## ConversationBufferMemory

The simplest option. It stores every message in the conversation — human and AI — and injects the full history into each new prompt.

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain
from langchain_openai import OpenAI

llm = OpenAI(temperature=0.7)
memory = ConversationBufferMemory()

conversation = ConversationChain(llm=llm, memory=memory, verbose=True)

conversation.predict(input="Hi, I'm Varun.")
conversation.predict(input="What's my name?")
```

The second call works. The model knows your name because the first exchange is still sitting in the buffer, prepended to the new prompt.

The catch: the buffer grows with every message. Long enough conversations will push you past the model's context window and start costing real money per call. Fine for short sessions, fragile for anything open-ended.

## ConversationBufferWindowMemory

Same idea as the buffer, but with a sliding window. You set `k` — the number of recent exchanges to keep — and anything older than that gets dropped.

```python
from langchain.memory import ConversationBufferWindowMemory

memory = ConversationBufferWindowMemory(k=5)
```

With `k=5`, the model sees the last five human-AI pairs and nothing before that. It's predictable and token-efficient. The tradeoff is that older context disappears hard — the model won't remember what you said in turn one once you're past turn six.

Good fit for chat interfaces where recent context is what matters.

## ConversationSummaryMemory

Instead of storing raw messages, this one compresses older parts of the conversation into a running summary using an LLM call.

```python
from langchain.memory import ConversationSummaryMemory
from langchain_openai import OpenAI

llm = OpenAI(temperature=0)
memory = ConversationSummaryMemory(llm=llm)
```

The summary gets prepended to the prompt in place of the full history. Older exchanges don't vanish — they get distilled. A 20-turn conversation might compress down to a paragraph.

The cost: you're making extra LLM calls to generate summaries. For very short conversations, that's overhead with no benefit. But for long sessions where you genuinely need the model to know what happened early on, this is often the right choice.

## ConversationSummaryBufferMemory

A hybrid. It keeps recent messages verbatim (like the buffer) and summarizes everything older than a token threshold.

```python
from langchain.memory import ConversationSummaryBufferMemory
from langchain_openai import OpenAI

llm = OpenAI(temperature=0)
memory = ConversationSummaryBufferMemory(llm=llm, max_token_limit=500)
```

Once the raw message history crosses `max_token_limit` tokens, older messages get rolled into a summary. Recent messages stay as-is. You get precise recent context and a compressed view of history — without blowing up your token count.

This is the one worth reaching for in production chatbots that need to handle both short and long conversations gracefully.

## ConversationEntityMemory

A different approach entirely. Instead of storing a transcript or a summary, it extracts entities from the conversation — names, places, products, concepts — and tracks what the model knows about each one.

```python
from langchain.memory import ConversationEntityMemory
from langchain_openai import OpenAI

llm = OpenAI(temperature=0)
memory = ConversationEntityMemory(llm=llm)
```

If you mention "Priya is the team lead on the backend project," the memory stores that fact about Priya. Later, when you ask "what's Priya working on?", the model has the entity context to answer correctly — even if the original message was 50 turns ago.

Useful when your chatbot needs to track facts about specific people or things across a long session, not just the conversational flow.

## Picking one

| Memory type | Keeps | Best for |
|---|---|---|
| `ConversationBufferMemory` | Full transcript | Short sessions, prototypes |
| `ConversationBufferWindowMemory` | Last `k` exchanges | Chat UIs with short context needs |
| `ConversationSummaryMemory` | LLM-generated summary | Long sessions, token budget matters |
| `ConversationSummaryBufferMemory` | Recent messages + summary of older ones | Production bots, variable session lengths |
| `ConversationEntityMemory` | Facts about named entities | Assistants tracking people, places, things |

Start with `ConversationBufferMemory` when you're prototyping — it's zero config and easy to debug. Move to `ConversationSummaryBufferMemory` when you start hitting token limits or dealing with sessions that run long. Use `ConversationEntityMemory` when your use case is genuinely about tracking facts across a conversation, not just maintaining thread.

Every memory type plugs into `ConversationChain` (or any chain that accepts a `memory` argument) the same way. The interface doesn't change — just what gets stored and how.