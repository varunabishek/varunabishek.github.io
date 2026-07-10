Title: Agents vs. Chains in LangChain: Who's Actually Driving?
Date: 2026-07-10
Category: GenAI
Tags: GenAI, LLM, LangChain, Agents
Slug: Who's-Actually-Driving?


*A chain follows the map you drew. An agent draws its own.*

LangChain gives you two ways to wire an LLM into a workflow: chains and agents. Both connect prompts, tools, and models into something useful, but the control flow works in opposite directions.

## Chains: fixed routes

A chain is a sequence of steps you define up front. Step one calls the model, step two parses the output, step three feeds it into a database lookup, step four formats a response. The order is fixed at build time, and the LLM has no say in what happens next.

**LLMChain** — The simplest building block: a prompt template plus a model call. Input goes in, text comes out.

**SequentialChain** — Runs multiple chains back to back, passing outputs forward as inputs. Good for multi-stage pipelines like "summarize, then translate, then extract keywords."

**RouterChain** — Picks one of several chains to run based on the input, but the routing logic itself is deterministic code, not a model deciding mid-task.

Chains are predictable. You can trace exactly what will happen before you run anything, and that makes them easy to test and debug.

## Agents: the model picks the path

An agent flips the control. Instead of a hardcoded sequence, the LLM gets a set of tools and a goal, and it decides at each step what to do next: which tool to call, what input to give it, and when it's done.

**Tool** — A function the agent can call: a search API, a calculator, a SQL query runner, a custom Python function. The agent decides when and how to use it.

**AgentExecutor** — The runtime loop that lets the agent think, act, observe the result, and think again, until it reaches a final answer.

**ReAct pattern** — A common agent style where the model alternates between reasoning ("I need to look up the current stock price") and acting (calling the tool that does it).

This loop keeps going until the agent decides it has enough information to answer, or it hits a step limit you set as a safeguard.

## Where the two actually diverge

The core difference isn't the tools or the prompts. It's who decides the next step.

- In a chain, you decide. The path is baked into your code.
- In an agent, the model decides. The path emerges while it runs.

That single distinction cascades into everything else. Chains are deterministic and fast because there's no reasoning overhead between steps. Agents are flexible but slower and less predictable, since each decision costs a model call and a wrong turn can send the whole task off track.

## When to use which

Reach for a chain when the task has a known shape: extract fields from a document, translate text, summarize an article. You already know the steps, so let code handle the sequencing and save the model calls for the parts that actually need judgment.

Reach for an agent when the task's shape depends on the input: answering open-ended questions that might need a web search, a calculation, or a database query depending on what's asked. You can't write that branching logic by hand because you don't know in advance which branch applies.

A lot of production systems end up doing both — a chain that occasionally calls out to an agent for the one step that needs open-ended tool use, then hands control back to the fixed sequence. Pick the smallest amount of agency the task actually requires. More autonomy means more places for things to go wrong.