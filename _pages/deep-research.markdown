---
layout: page
title: Deep Research 101
permalink: /deep-research/
includelink: false
comment: false
---

Everything I have learnt about deep research are from the following resources:

1️⃣ [LangChain's deep research from scratch](https://github.com/langchain-ai/deep_research_from_scratch)

2️⃣ [LangChain's deep agents from scratch](https://github.com/langchain-ai/deep-agents-from-scratch)

3️⃣ [renschi's](https://github.com/renschni) [Manus agent report](https://gist.github.com/renschni/4fbc70b31bad8dd57f3370239dccd58f)


**The core idea is to:**

✅ Have a "main" agent, that is provided with a bigger goal which it needs to achieve using specialised sub-agents 

✅ A set of sub-agents who have smaller goals and tools they need, to help them achieve their individual goals - These sub-agents are used by the main agent as tools

✅ Breakdown the bigger goal into multiple smaller tasks - A list of ToDos or plans or task items. The main agent is tasked to think and breakdown the bigger goals into specific (nearly) atomic tasks that can be executed independently

✅ Having access to a file system tool that can read-write the ToDos on demand rather than putting everything in prompt - As the complexity grows, the information you want to put in the prompt also grows - This is exactly we need to keep check of

✅ Provide the main agent with a sub-agent selection tool to help determine which sub-agent is good at which task - The trick is to treat the response from the sub-agents as `ToolMessage` rather than `AIMessage`. So when the work done by the sub-agents are sent back to the main agent, to the main agent they are simply output from tool calls

✅ Cleverly written [**ReAct** prompts](https://smith.langchain.com/hub/hwchase17/react) and efficient context engineering [**read this**](https://simonwillison.net/2025/jun/27/context-engineering/) and [**this**](https://www.philschmid.de/context-engineering) - Create as much context for individual agents, remove anything not required

✅ ToDos can be read from file system and put into context as required, RAG/knowledge graph based data retrieval also helps to identify key information required for the particular agent in action and providing it the particular information it needs (nearly zero-noise contenxt - as LLMs usually hallicinate after 33K tokens)

✅ The "main" agent finally aggregates all the results from all the sub-agents and creats the final research report in one shot

I have used this pattern for a quite complicated research-like use case (with GPT 4.1 mini).. 

The results were amazing! 

🔥 On top of this core idea of deep research, I had used periodic summarization of messages exchanged, context tailoring and evaluation of report using the LLM-as-judge technique.

🔥 LLM-as-judge could simply be done by having a specialist agent with "evaluator" role that knows your evaluation criteria (may be with examples), and it evaluates the content provided with discrete labels like POOR, AVERAGE, GOOD or VERY GOOD. Then your conditional acceptance may go ahead with GOOD and VERY GOOD reports, and send POOR and GOOD reports back to the main agent to re-do the research. The trick is to configure your evaluator agent to write a feedback for substandard reports, which you can put into the context of the main or sub-agents in the next pass.

Give it a try.
