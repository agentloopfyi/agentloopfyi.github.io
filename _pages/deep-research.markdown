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

```python
from langgraph.prebuilt import create_react_agent

def get_deep_research_agent():
  deep_research_agent = create_react_agent(
    model=get_llm(),
    tools=[sub_agent_selection_tool],
    response_format=CustomStructuredOutput,
  )

  return deep_research_agent
```

✅ A set of sub-agents who have smaller goals and tools they need, to help them achieve their individual goals - These sub-agents are used by the main agent as tools

```python
from langgraph.prebuilt import create_react_agent
from langgraph.graph.state import CompiledStateGraph

def get_sub_agent_1():
  return create_react_agent(
    model=get_llm(),
    tools=[...],
    prompt=SUB_AGENT_1_PROMPT,
  )

def get_sub_agent_2():
  return create_react_agent(
    model=get_llm(),
    tools=[...],
    prompt=SUB_AGENT_2_PROMPT,
  )

def get_sub_agent_3():
  return create_react_agent(
    model=get_llm(),
    tools=[...],
    prompt=SUB_AGENT_3_PROMPT,
  )

def get_sub_agent_4():
  return create_react_agent(
    model=get_llm(),
    tools=[...],
    prompt=SUB_AGENT_4_PROMPT,
  )

def get_sub_agent_5():
  return create_react_agent(
    model=get_llm(),
    tools=[...],
    prompt=SUB_AGENT_5_PROMPT,
  )

SUB_AGENT_TYPE = Literal[
  "sub_agent_1",  # Specialist in task A
  "sub_agent_2",  # Specialist in task B
  "sub_agent_3",  # Specialist in task C
  "sub_agent_4",  # Specialist in task D
  "sub_agent_5",  # Specialist in task E
]

subagents: Dict[SUB_AGENT_TYPE, CompiledStateGraph] = {}

subagents["sub_agent_1"] = get_sub_agent_1()
subagents["sub_agent_2"] = get_sub_agent_2()
subagents["sub_agent_3"] = get_sub_agent_3()
subagents["sub_agent_4"] = get_sub_agent_4()
subagents["sub_agent_5"] = get_sub_agent_5()
```

✅ Breakdown the bigger goal into multiple smaller tasks - A list of ToDos or plans or task items. The main agent is tasked to think and breakdown the bigger goals into specific (nearly) atomic tasks that can be executed independently

✅ Having access to a file system tool that can read-write the ToDos on demand rather than putting everything in prompt - As the complexity grows, the information you want to put in the prompt also grows - This is exactly we need to keep check of

✅ Provide the main agent with a sub-agent selection tool to help determine which sub-agent is good at which task - The trick is to treat the response from the sub-agents as `ToolMessage` rather than `AIMessage`. So when the work done by the sub-agents are sent back to the main agent, to the main agent they are simply output from tool calls

```python
from langchain_core.messages import HumanMessage, ToolMessage
from langchain_core.tools import InjectedToolCallId, tool
from langgraph.types import Command

SUB_AGENT_SELECTION_TOOL_DESCRIPTION = """
Delegate a task to a specialized sub-agent with isolated context.
Available sub-agents are: sub_agent_1, sub_agent_2, sub_agent_3, sub_agent_4 and sub_agent_5

Args:
    - content: Content (atleast 5 sentences, more the better) extracted from input given to you, which is required for this sub-agent call
    - which_sub_agent: Exactly one sub-agent name that you can delegate to, in order to fulfil the current task
    
"""

SUB_AGENT_TEMPLATE = """
Content you need to use to do your job:
{content}

"""

@tool(description=SUB_AGENT_SELECTION_TOOL_DESCRIPTION)
def sub_agent_selection_tool(
  content: str,
  which_sub_agent: str,
  tool_call_id: Annotated[str, InjectedToolCallId]):
  
  LOGGER.debug(f"<!-- Tool called; sub_agent called: {which_sub_agent}")
  
  sub_agent = subagents[which_sub_agent]
  result = sub_agent.invoke(
    {
      "message": [
        HumanMessage(
          content=SUB_AGENT_TEMPLATE.format(
            content=content,
          )
        )
      ]
    }
  )

  return Command(
    update={
        "messages": [
            ToolMessage(result["messages"][-1].content, tool_call_id=tool_call_id)
        ],
    }
  )
```

✅ Cleverly written [**ReAct** prompts](https://smith.langchain.com/hub/hwchase17/react) and efficient context engineering [**read this**](https://simonwillison.net/2025/jun/27/context-engineering/) and [**this**](https://www.philschmid.de/context-engineering) - Create as much context for individual agents, remove anything not required

✅ ToDos can be read from file system and put into context as required, RAG/knowledge graph based data retrieval also helps to identify key information required for the particular agent in action and providing it the particular information it needs (nearly zero-noise contenxt - as LLMs hallucinate when they have to juggle with a lot of different information). [More on ToDos.. the LangChain way](https://www.youtube.com/watch?v=dwvhZ1z_Pas).

✅ The "main" agent finally aggregates all the results from all the sub-agents and creates the final research report in one shot

```python
DEEP_RESEARCH_PROMPT = """
You are an expert in this and that. You have access to specialist sub-agents to delegate your work.
Each sub-agent will need specific sections from the input porvided to you, you need to extract specific sections for each sub-agent, provide the extract to the corresponding sub-agent during delegation and get the work done.

Available agents for delegation are:

- sub_agent_1: Expert in task A
- sub_agent_2: Expert in task B
- sub_agent_3: Expert in task C
- sub_agent_4: Expert in task D
- sub_agent_5: Expert in task E

You are provided with the input below, based on which you need to do your job. 

<Input>
{input}
</Input>

The input has been evaluated by an expert evaluator as {eval} with the following reasons:

<Reason>
{eval_reason}
</Reason>

**Your goal:**
Blah blah blah

**Your tasks:**
1. Delegate work to each of the agents based on their specializations by providing the content they need to do their job. 
   Think and extract (atleast 5 sentences, more the better) specific sections from the requirement summary, as applicable for each sub-agent based on their specialization.   
2. Do task A 
3. Do task B
4. Do task C
5. Do task D
6. Do task E
7. Combine output exactly as returned by all the sub-agents into one single report. 
8. Add a summary at the end of the report.
9. Add your recommendation and commentary on the full report.

Your output must be the exact report only; NO additional sentences, follow-up questions or extra information should be there.

"""

llm_response = get_deep_research_agent().invoke(
  {  
    "messages": [
      HumanMessage(content=DEEP_RESEARCH_PROMPT.format(
        input=state["input"],
        eval=state["eval"], # Discrete values like, VERY GOOD, GOOD, OK, etc
        eval_reason=state["eval_reason"],) # Comments on quality of the input, could be from an evaluator agent obtained before
      )
    ]    
  }
)
```

I have used this pattern for a quite complicated research-like use case (with GPT 4.1 mini).. 

The results were amazing! 

🔥 On top of this core idea of deep research, I had used periodic summarization of messages exchanged, context tailoring and evaluation of report using the LLM-as-judge technique.

🔥 LLM-as-judge could simply be done by having a specialist agent with "evaluator" role that knows your evaluation criteria (may be with examples), and it evaluates the content provided with discrete labels like POOR, AVERAGE, GOOD or VERY GOOD. Then your conditional acceptance may go ahead with GOOD and VERY GOOD reports, and send POOR and GOOD reports back to the main agent to re-do the research. The trick is to configure your evaluator agent to write a feedback for substandard reports, which you can put into the context of the main or sub-agents in the next pass.

Give it a try.
