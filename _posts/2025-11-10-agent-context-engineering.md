---
layout: post
show_meta: true
title: Context engineering in agent
header: Applying context engineering to an agent
date: 2025-11-10 00:00:00
summary: How to apply context engineering to an agent
categories: python gemini agent context-engineering
author: Chee Yeo
---

[Lang-chain article on context engineering]: https://blog.langchain.com/the-rise-of-context-engineering/


[Own your context window]: https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md


When we interact with an LLM via an agent, we tend to use prompt templates with some dynamic data filled in before invoking the LLM. The issue with using static prompts is that the context within the prompt is fixed whereas in the real world, the context of each prompt should be unique and scoped to each particular agent call.

The term `context engineering` is used to describe a system or process of generating a unique context for each agent invocation. From the lang-chain blog, this is defined as the output of a system that runs before the main LLM call. The context returned is dynamic, created on the fly and tailored to the immediate task at hand. More importantly, its about ensuring that the right tools and information are available to the agent at the right time. This could be in the form of historical data via a knowledge base or additional capabilities via tools and function calls to retrieve more information.

To implement context engineering, from the definitio given above, we can identity the following core components:
* Tool use, function calling
* Short term memory ( current history )
* Long term memory ( mem0 )
* Retrieval ( fetching information via RAG )



( discuss the code for the MCP server repo on context engineering )