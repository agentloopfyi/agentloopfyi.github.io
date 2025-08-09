---
layout: post
comments: false
title:  "Prompt engineering"
excerpt: "Prompt engineering is the practice of crafting clear, specific instructions to get the best possible responses from AI models. Effective prompts typically include detailed context, specific examples of desired outputs, and clear formatting requirements to guide the AI's response. "
date:   2025-06-22 10:00:00
---
Getting meaningful responses from AI tools like ChatGPT isn't just about asking questions, it's about asking the right questions. Prompt engineering is the art of crafting inputs that unlock AI's full potential.

### What makes a great prompt?
Think of prompts as detailed instructions for a highly capable but literal assistant. The more specific and structured your request, the better the output. This means considering tone, context, and clear constraints.

Want better results? Try role-based prompting: "You are a marketing expert and create..." or use examples to show exactly what you need. Setting boundaries works too. For example, "explain this concept in simple terms under 100 words."

### Why it matters?
AI models don't truly "understand" like humans do. They recognize patterns and calculate probabilities based on training data. A vague prompt leads to generic responses, while a well-crafted one delivers precisely what you need. No model retraining required for that.

## Popular prompt patters:

### Zero shot prompting

You simply ask the model a question. Like *What is the capital of France?* You didn't provide any example. This works fine for generic and simple questions that the model can answers based on its knowledge.
<hr />

### One shot prompting

You give *one* simple and clear example, and then ask a question to the model. Based on the example provided, the model will try togenerate an answer. This comes handy when you want to model to simple *do* something very specific and avoid the *randomness*.

```text
Categorize the input provided to you in a relevant topic as example shown below.

Text: Canada is hosting the G7 summit in 2025.
Category: Geo-politic

Text: NIFTY opens low.
Category: 

```
<hr />

### Few shot prompting

You give *multiple* simple and clear examples, and then ask a question to the model. Based on the examples provided, the model will try togenerate an answer. For tasks that might lead to different output based on specific variations in the input, but not really too complicated, few shot prompting works really well.

```text
A is 1
B is 2
C is 3
and so on...

CAT is 3.1.20
DOG is 4.15.7
RABBIT is 18.1.2.2.9.20

What is TIGER?

```
<hr />

### Chain of Thoughts prompting

Ever notice how AI sometimes jumps to conclusions without showing its work? Chain of Thought (CoT) prompting fixes that by teaching AI to think out loud, step by step.

#### Teaching AI to think like humans (Simulating "thinking")
Instead of demanding instant answers to complex problems, CoT prompting mirrors how we naturally solve challenges, breaking them into manageable pieces. This technique transforms AI from a magic 8-ball into a thoughtful problem-solver that shows its reasoning process.

#### Two powerful approaches
You can guide AI thinking in two ways. Few-shot CoT involves providing examples with detailed reasoning steps. Zero-shot CoT is even simpler, Just add "Let's think step by step" to your prompt.

See the Difference:

Without CoT: 
```text
Question: "If a train travels 60 km/h for 2 hours, how far does it travel?"
Answer: 120 km.
```

With CoT: 
```text
Question: "If a train travels 60 km/h for 2 hours, 
how far does it travel? Let's think step by step."

Answer: The train travels at 60 km/h for 2 hours.
 Using Distance = Speed × Time: 60 × 2 = 120 km. 
 The train travels 120 km.
```

#### Why it works?
CoT prompting shines in arithmetic, logic puzzles, and reasoning tasks. By forcing the AI to show its work, you catch errors early and build trust in the results. The structured approach dramatically improves accuracy on complex problems, turning AI into a reliable thinking partner.
<hr />

### ReAct prompting

#### When AI thinks AND acts...

What if AI could not only think through problems but also take action to find answers? ReAct (Reasoning and Acting) prompting makes this possible by combining internal reasoning with real-world interactions.

#### Beyond pure thinking
Traditional AI prompting relies on the model's existing knowledge to solve problems. ReAct takes it further by creating a dynamic loop where AI alternates between thinking about the problem and taking concrete actions-like searching the web, running calculations, or accessing databases.

#### The Think-Act-Observe cycle
ReAct follows a powerful three-step pattern:

```text
Thought: AI analyzes what it needs to know
Action: AI takes a specific step (search, calculate, query)
Observation: AI processes the results and decides next steps
```

This cycle repeats until the AI reaches a solid conclusion.
See It in Action

```text
Question: How much tariff did the US impose on India?
Thought: I need to look up the tariff the US imposed on India.
Action: Search["current tariff on India as imposed by the US"]
Observation: The search result says 26% is the current tariff.
Thought: Now I have the answer based on current information.
Answer: 26%.
```

#### Why it matters?
ReAct prompting reduces AI hallucinations by grounding responses in real data. It creates transparency and you can see exactly how AI reached its conclusion. This approach powers advanced AI agents that can interact with tools, making them incredibly valuable for complex, multi-step tasks requiring current information.
<hr />

### Tree of Thoughts prompting

ℹ️ *work in progress*
<hr />

### Divide and conquer prompting

ℹ️ *work in progress*
