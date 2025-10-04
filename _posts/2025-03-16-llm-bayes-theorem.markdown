---
layout: post
comments: false
title:  "Large Language Models and Bayes' Theorem"
date:   2025-03-16 10:00:00
---

Large Language Models are predicting the next token. But how do we influence this process?

For a given sequence of tokens, LLMs are computing what could be the next most probable token.

Then, the "previous input sequence of tokens" plus "the predicted output token" becomes the next input..

..and same way, again the next most probable token is computed..

..and this repeats!

🎲 It's all probability. But it's stochastic (random).

Having the same input does not guarantee the same output. Also, a little change in the input, will most likely change the output altogether.

How do we influence this process to get us the most likely output? There are many low level techniques, but as developers, we often rely on prompts. But how does a prompt help really?

If you think about it, it has conceptual similarities with Bayes' Theorem. May not be exactly like it, but there is a striking resemblance.

Bayes' Theorem is used in computing the probability of event A occurring, given event B has occurred, mathematically defined as:

**P(A \| B)**

When you write a prompt, "You are a helpful assistant who provides information on X, your job is to do Y"..

..you have defined that event B, that is "You are a helpful assistant who provides information on X", has occurred, and then you ask the most likely Y, that is event A.

Mathematically, what you're asking is:

**P(Y \| You are a helpful assistant who provides information on X) = ?**

This way, the prompt helps the LLM predict better, as the LLM is now trying to predict something based on something that has already happened. Therefore, the "most likely output" (event A), is influenced by a preset context (event B), which helps the LLM to compute the probability (i.e., likelihood of the output) in a somewhat "guided" way.

Intriguing, isn't it?

You will not look at prompts the same way.
