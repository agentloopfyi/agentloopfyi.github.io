---
layout: post
comments: false
title:  "Human In the Loop using LangGraph"
date:   2025-12-03 10:00:00
---

📌 **What is HITL?**

When you have introduced quite a lot of autonomy in your workflows but still there are some actions that are best taken care by humans, then you can introduce an "interrupt".. that brings a human action into the agent loop. The agent will wait, until a human takes and action, by persisting the state of the workflow till that point, and then resume based on the action that the human has taken. The action taken by the human could be a response with a semantic meaning that could be resolved to an action, or you can provide choices like, "Yes", "No", "Approve", "Reject", etc, and resume the graph from the interrupt.

📌 **How HITL internally works?**

I am personally a LangGraph fan. HITL interrupts are raised in LangGraph by using the `interrupt` function from the `langgraph.types` module.

You call this function in the graph node that you need to specifically keep for raising the interrupt.

Remember, ideally the `interrupt` function call must be the first instruction inside the node. You may have some variable intialization before that, but dont do anything that might have a side effect. Because, when the graph resumes from the interrupt, then it does not resume from the particular line of code in your interrupt node function where you called `interrupt`, rather by executing the interrupt node itself from the beginning of the function. Think of interrupts as an exception raised by LangGraph, having the state persisted to that point and then resuming from that point itself when human takes an action.

🚀 Therefore, you cannot raise HITL interrupts without checkpointing. Good to use in-memory checkpointing in examples, but use Redis or Postgres checkpointing in live environments.

📌 **Working example**

Simple example showing...

1️⃣ Graph starts executing.

2️⃣ Raises interrupt.

3️⃣ Human resumes from choices of yes, no and may be.

**Raise an HITL interrupt**

```python
from typing import Literal, Tuple
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langchain_core.messages import AIMessage, HumanMessage
from langgraph.checkpoint.memory import InMemorySaver 

# Agent state is your custom LangGraph state

async def starting_node(state: AgentState) -> AgentState:
  # do stuff
  # return {...}
  
def interrupt_node(state: AgentState) -> Command[Literal["ending_node"]]: 
  banner = "Do you want to continue?" # The message you want to show the user when raising the interrupt
  human_input: str = interrupt({"banner": banner, "resume_choices": ["YES", "NO", "MAY BE"]}) # Choices available to the user to resume from the interrupt

  next_node = 'yes_node'
  if human_input.lower() == 'yes':
    next_node = 'yes_node'
  elif human_input.lower() == 'no':
    next_node = 'no_node'
  else:
    next_node = 'may_be_node'

  return Command(
    goto=next_node, # commanding the graph which node to go after resuming
    # You should also update the messages to keep track of the interrupt message as AI Message and the choice selected by human as Human Message
    # This helps maintaining the flow of messages in the same AI, Human, AI, Human... sequence
    update={
      "messages": [
        AIMessage(content=banner), # The messages shown when raising the interrupt
        HumanMessage(content=human_input), # YES NO or MAY BE, which ever choice human selected
      ]
    },
  )
    
async def yes_node(state: AgentState) -> AgentState:
  # take YES actions
  # return {...}
  pass # for now

async def no_node(state: AgentState) -> AgentState:
  # take NO actions
  # return {...}
  pass # for now

async def may_be_node(state: AgentState) -> AgentState:
  # take MAY BE actions
  # return {...}
  pass # for now

builder = StateGraph(AgentState)
builder.add_node(starting_node)
builder.add_node(interrupt_node)
builder.add_node(yes_node)
builder.add_node(no_node)
builder.add_node(may_be_node)

builder.add_edge(START, "starting_node")
builder.add_edge("starting_node", "interrupt_node")
builder.add_edge("interrupt_node", "yes_node")
builder.add_edge("interrupt_node", "no_node")
builder.add_edge("interrupt_node", "may_be_node")
builder.add_edge("yes_node", END)
builder.add_edge("no_node", END)
builder.add_edge("may_be_node", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

```

Note, when we call the `interrupt` function, we pass a dict with keys `banner` and `resume_choices`.

Here you can pass any value, this value will be used by LangGraph while throwing the HITL exception. You can very well pass a string. Structuring this in a dict helps you group different values with readable names. For example, `banner` is the value that we will show the user while raising the interrupt, and the set of choices in `resume_choices` are available to the user to resume from there.

Somewhere in your app, where you invoke the graph with state data, you will do something like...

```python
state = await graph.ainvoke(current_state, config=config)
```

Then, in order to know whether this flow generated an interrupt, you need to do something like...

```python
if "__interrupt__" in state:
  # Interrupt raised
  interrupt_message = state["__interrupt__"][0].value
  ai_message = str(interrupt_message["banner"]).strip(),
  resume_choices = list(interrupt_message["resume_choices"])
  is_interrupted = True
  # Then carry this variables back to the UI, this ai_message is the banner that you constructed while raising the interrupt
  # Carry the resume_choices along with, each item in this list can be ren  dered on your UI as clickable components to later resume the graph by calling an API
  # Note we are sending an additional boolean
else:
  # Interrupt not raised
  # do stuff
  pass # for now
```

Send this `is_interrupted` flag to the UI to tell this is not a regular response, rather an HITL response.

🚀 Therefore, when the graph raises an interrupt, the state returned by the `ainvoke` call (or `invoke` call) contains a special key by the name of `__interrupt__`.

**Resume from interrupt**

Based on the `is_interrupted` flag we sent earlier, the UI while making the next API call when the user clicks one of the resume choices, can tell the API using a flag like `resuming_from_interrupt` that this is not a regular API call.. we are resuming from an HITL interrupt.

```python
if request.resuming_from_interrupt is True:
  state = await graph.ainvoke(
    Command(resume=request.human_message), config=config
  )
else:
  state = await graph.ainvoke(current_state, config=config)
```

🚀 We resume from HITL interrupts using `Command` in the `ainvoke` call (or `invoke` call) on the graph. Resume using the choice clicked by the human. Send the exact values, yes, no or may be.

That's pretty much you need to build a working HITL workflow. Happy learning!

🚀 Ofcourse, software is eventually about abstractions. If you want to use the HITL middleware, look [here](https://docs.langchain.com/oss/python/langchain/middleware/built-in#human-in-the-loop).
