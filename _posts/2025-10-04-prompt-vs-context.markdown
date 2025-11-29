---
layout: post
comments: false
title:  "Prompt vs Context Engineering"
date:   2025-10-04 10:00:00
---

[1] [Context engineering by Sam Wilson](https://simonwillison.net/2025/jun/27/context-engineering/)

[2] [Context engineering by Phil Schmid](https://www.philschmid.de/context-engineering)

LLMs hallucinate after ~30K tokens (I read this somewhere, I don't know the exact number.. But LLMs do halucinate as the token input size grows).

Also, more tokens mean more money spent - And doesn't necessarily mean better answers. 

While prompts have been doing wonders with techniques like ReAct, chain of thoughts, etc, it is clear as interaction with LLMs increase throughout the session or workflow, it is not really a practical option to put more and more messages and information in the prompt.

Our intention must not only be to stay within the context window, but also keep it as concise as possible, making sure only the relevant and necessary information are sent to the LLM that it needs to do its job.

One technique that worked well for me in few of the agentic AI use cases that indeed went to production, was having "phases".

A workflow is a sequence of steps, where in each step we invoke the LLM with a prompt and state. Divide your workflow into phases.

Let's say, there are 7 steps, which can be logically grouped into 2+2+3 steps. In steps one and two, the workflow perhaps is interacting with the user (as a conversation assistant) and exchanging lots of messages.

At the end of step two, summarize all the messages exchanged with the user as notes on a "scractchpad". 

You can use an intermediate "helper agent" that specifically uses a prompt to summarize the conversation into the scratchpad by compressing the conversation history while preserving the intent and important information collected.

Then the scratchpad becomes the input to the third step (not the entire conversation history). Step three onwards, you are not carrying around the load of the full conversation and saving yourself lots of tokens.

Also, later, you can always ask an agent to expand the "scratchpad" content into a report or write an explanation.

This is truly "generative" as you may think, that we are compressing the conversation history onto a scratchpad (like a latent space, preserving the intent, sense and meaning), and then we generate entirely new content from the scratchpad by expanding it, and the new content has the same intent, sense and meaning.

Scratchpads are also good opportunities to inject additional information that might provide relevant context to the LLMs in steps afterwards.

For example, you can search and embed document content, rules, examples, etc, into the scratchpad that will help the LLM later.
