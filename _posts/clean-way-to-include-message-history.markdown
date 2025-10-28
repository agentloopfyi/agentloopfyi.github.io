---
layout: post
comments: false
title:  "Clean (LangGraph) way to include message history into prompt when making LLM API calls"
date:   2025-10-28 10:00:00
---

LangGraph is my go-to framework for building graph-based agentic workflows involving multiple agents.

Oftentimes, you need to facilitate a conversation between the user and the LLM and make sure the conversation history is retained in every subsequent calls.

Therefore every time you make a call the the LLM API, in one way or the other you need to include the messages history in the payload, as a sequence of human and AI messages.

In LangGraph we keep messages in the state, which might look something like:

```python
# MessagesState as defined in langgraph.graph
# class MessagesState(TypedDict):
#  messages: Annotated[list[AnyMessage], add_messages]

# Inherit MessagesState in your app's state class to get access to "messages" and the reducer
class AgentState(MessagesState):
 state_attr: str

```

Then, just before calling your agent, which is basically an LLM call with a specific prompt (and tools if required), you create the system prompt as below:

```python
from langchain_core.messages import (
    SystemMessage,
    get_buffer_string,
)

def agent_node(state: AgentState) -> AgentState:
  AGENT_PROMPT = """You are an expert in this and that.
  
  **Your tasks:**
  - blah
  - blah
  
  These are the messages that have been exchanged so far with the stakeholder asking for different specifications:
  <Messages>
  {messages}
  </Messages>
  """
  
  SYSTEM_MESSAGE = SystemMessage(
    content=AGENT_PROMPT.format(
      messages=get_buffer_string(state["messages"]),
    )
  )
  
  llm_response = get_agent().invoke([SYSTEM_MESSAGE])

  # Update state
  return {
    "messages: [AIMessage(content=result["messages"][-1].content)]
  }

```

Using `get_buffer_string` with include the conversation in a way that the LLM usually can interpret better as a series of human and AI messages.

The system prompt will look like:

```text
You are an expert in this and that.
  
**Your tasks:**
- blah
- blah

These are the messages that have been exchanged so far with the stakeholder asking for different specifications:
<Messages>
Human: ....
AI: ....
Human: ....
AI: ....
Human: ....
AI: ....
Human: ....
AI: ....
</Messages>

```

As a developer, you get a cleaner approach, that you can log and visually understand as simple text in sequence with role, rather than logging the list of message objects.
