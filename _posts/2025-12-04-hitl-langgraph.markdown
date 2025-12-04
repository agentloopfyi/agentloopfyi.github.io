---
layout: post
comments: false
title:  "Human In the Loop with LangGraph"
date:   2025-12-04 10:00:00
---

🚧 **Work in progress**

### Implementing human-in-the-loop (HITL) in LangGraph

#### What is HITL?

#### Why do we need HITL?

#### How HITL internally works?

#### Working example

```python
from typing import Literal, Tuple
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langchain_core.messages import AIMessage, HumanMessage

# Agent state is your custom LangGraph state

async def starting_node(state: AgentState) -> AgentState:
  # do stuff
  # return {...}
  
def interrupt_node(state: AgentState) -> Command[Literal["ending_node"]]: 
  banner = "Do you want to continue?" # The message you want to show the user when raising the interrupt
  human_input: str = interrupt({"banner": banner, "resume_choices": ["YES", "NO"]}) # Choices available to the user to resume from the interrupt

  return Command(
    goto="ending_node",
    update={
      "messages": [
        AIMessage(content=banner), # The messages shown when raising the interrupt
        HumanMessage(content=human_input), # YES or NO, according to resume choices
      ]
    },
  )

async def ending_node(state: AgentState) -> AgentState:
  # do stuff
  # return {...}

builder = StateGraph(AgentState)
builder.add_node(starting_node)
builder.add_node(interrupt_node)
builder.add_node(ending_node)

builder.add_edge(START, "starting_node")
builder.add_edge("starting_node", "interrupt_node")
builder.add_edge("interrupt_node", "ending_node")
builder.add_edge("ending_node", END)

graph = builder.compile()

```

#### Conclusion
