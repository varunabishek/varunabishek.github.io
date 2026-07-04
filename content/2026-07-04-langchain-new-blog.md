Title: The Five Building Blocks of LangChain
Date: 2026-07-04
Category: GenAI
Tags: GenAI, AI-agents, LLM, agentic-systems, design-patterns, reliability
Slug: The-Five-Building-Blocks-of-LangChain




Most people's first LangChain project looks the same. They copy a snippet, swap in their API key, run it, and get a working chatbot in about ten minutes. Then they try to add memory, or connect a second tool, or make the thing reason through a multi-step task, and the whole mental model falls apart. The tutorial made it look like magic. It wasn't magic. It was five components working together, and nobody explained what each one actually does.

So let's fix that. Five pieces: Models, Prompts, Chains, Memory, Agents. Once you know what each one is for, LangChain stops feeling like a black box and starts feeling like a toolkit.

## Models: the engine, not the car

A model in LangChain is just a wrapper around a language model API — OpenAI, Anthropic, Cohere, a local Llama instance, whatever you're using. LangChain doesn't make the model smarter. It gives you one consistent interface so you can swap GPT-4 for Claude for a local model without rewriting your application logic.

That interface matters more than it sounds like it should. Say you build a product on OpenAI and six months later want to test whether Claude handles your use case better. Without an abstraction layer, that's a rewrite. With LangChain, it's often a one-line change.


pythonfrom langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4", temperature=0.7)
response = model.invoke("Explain quantum computing in one sentence.")
print(response.content)



Two things worth knowing early. First, temperature controls randomness — near 0 for consistent, deterministic output, higher for creative variation. Second, LangChain distinguishes between LLMs (plain text in, text out) and chat models (structured messages with roles like system, human, and AI). Almost everything you build today will use chat models, because that's what modern providers optimize for.

The model is the engine. On its own, it's not a product. You need somewhere to point it.

## Prompts: what you say determines what you get:

A prompt template is a reusable, parameterized instruction. Instead of hardcoding "Translate 'hello' to French," you write a template with a blank for the input, and reuse the same structure for any word or sentence.


pythonfrom langchain.prompts import ChatPromptTemplate

template = ChatPromptTemplate.from_template(
    "Translate the following text to {language}: {text}"
)

prompt = template.format_messages(language="French", text="Good morning")



This looks trivial, but it's where most of the actual engineering happens once you're past the demo stage. A prompt template can include few-shot examples, formatting rules, output constraints, persona instructions — anything that shapes how the model responds. Change the wording and the model's behavior changes with it, sometimes dramatically. Prompt engineering isn't a nice-to-have skill on top of LangChain. It's the interface layer between your intent and the model's output, and templates are what make that interface manageable instead of a pile of string concatenation.

## Chains: connecting the pipes

A chain links components together so output flows automatically from one step to the next. The simplest chain takes a prompt template, feeds it to a model, and hands you the result — no manual formatting, no manual API call.


pythonchain = template | model
result = chain.invoke({"language": "Spanish", "text": "See you tomorrow"})


That pipe operator is LangChain Expression Language, or LCEL. It reads left to right: take the template, pipe it into the model. You can extend that pipeline as far as you need — parse the output, pass it into a second prompt, feed that into a different model, chain a retrieval step before any of it happens.

Chains are where LangChain earns its name. A single API call to a model is rarely enough for a real application. You need to retrieve context, format it, generate a draft, check the draft against a rule, maybe regenerate. Each of those is a link. String enough of them together and you've built a pipeline that would otherwise be scattered across a dozen manual function calls — except now it's declarative, testable, and easy to swap pieces in and out of.

## Memory: making the model remember it's mid-conversation:

Here's a fact that surprises a lot of beginners: language models are stateless. Every API call is a clean slate. The model has no idea what you said two messages ago unless you explicitly send it that history again.

Memory in LangChain is the mechanism for doing that — storing past exchanges and re-injecting them into future prompts so the conversation feels continuous.


pythonfrom langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain

memory = ConversationBufferMemory()
conversation = ConversationChain(llm=model, memory=memory)

conversation.invoke("My name is Alex.")
conversation.invoke("What's my name?")



The second call works — the model answers "Alex" correctly — because memory quietly attached the first exchange to the second prompt. Without it, the model would have no way to know.

The catch is that conversation history isn't free. Every stored message gets resent with every new call, and tokens cost money and hit context limits. That's why LangChain offers several memory strategies beyond the basic buffer: summarizing old exchanges instead of storing them verbatim, keeping only the last N messages, or storing information as structured facts rather than raw transcript. Picking the right memory strategy is a real design decision, not a default you set once and forget.

## Agents: when the model decides what to do next

Chains follow a path you define in advance. Agents don't. An agent uses the language model to decide, at runtime, which action to take — which tool to call, in what order, based on what it learns from each step.

Give an agent access to a calculator, a search API, and a database query tool, and ask it a multi-part question. It might search for a fact, calculate something with the result, then query the database to verify it — choosing that sequence itself rather than following a route you hardcoded.


pythonfrom langchain.agents import create_react_agent, AgentExecutor
from langchain.tools import Tool

tools = [
    Tool(name="Search", func=search_function, description="Search the web for current information"),
    Tool(name="Calculator", func=calculate, description="Perform mathematical calculations"),
]

agent = create_react_agent(model, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools)

executor.invoke({"input": "What's the population of Japan divided by the population of the UK?"})


The agent recognizes it needs two numbers before it can do the division, searches for each one, then calculates the result — all without you writing that control flow by hand.

This is also where things get harder to predict. Agents can loop, misuse a tool, or take a route you didn't anticipate, because the reasoning happens inside the model rather than in code you wrote. Debugging an agent means reading through its intermediate steps to see where its logic diverged from what you expected, which is a different skill than debugging a fixed chain.

## Putting it together:

A useful way to see how these fit: models generate, prompts instruct, chains connect, memory persists, agents decide. A basic Q&A bot might need only a model and a prompt. A customer support assistant that remembers earlier tickets needs memory. A research assistant that has to search, calculate, and cross-reference on its own needs an agent.

Most real applications end up using several of these together — a chain with memory attached, or an agent that calls a chain as one of its tools. Once you can name what each piece is doing, that combination stops looking like a black box and starts looking like a set of parts you're choosing.