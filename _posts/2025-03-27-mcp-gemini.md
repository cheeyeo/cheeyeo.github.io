---
layout: post
show_meta: true
title: Example of using Gemini model with MCP
header: Example of using Gemini model with MCP
date: 2025-03-27 00:00:00
summary: How to intergate MCP server with Gemini model
categories: machine-learning mcp go-lang gemini
author: Chee Yeo
---

In a previous blog post, I explained what the ability of foundational models such as Gemini to interact with the external environment by invoking tools using function calling.

The Gemini API allows you to define a list of tools by defining a list of `genai.Tool` struct, each of which defines a function declaration:
```
geminiTools := []*genai.Tool{}

geminiTool := &genai.Tool{
    FunctionDeclarations: []*genai.FunctionDeclaration{{
        Name:        tool.Name,
        Description: *tool.Description,
        Parameters:  toolProperties,
    }},
}

geminiTools = append(geminiTools, geminiTool)

model := geminiClient.GenerativeModel("gemini-2.5-pro-preview-03-25")
model.Tools = geminiTools
```

The issue with this approach is that in order to use custom tools, we need to define them together with the gemini model code. 


( short desc of MCP )
A better approach is to use an architecture such as MCP which separates the development and hosting of the tools on its own server. We register each tool within the MCP server. Each tool has a name, description and a function handler that accepts the arguments passed in and returns a suitable response via the `mcp_golang.ToolResponse` type.

How do we link this to a gemini model? Recall the code snippet earlier where we can set a list of tools to be used by the model. The client for the MCP server can invoke a list of tools using `client.ListTools`. We can iterate over each tool and convert it into a suitable format of `genai.Tool` and register it with the gemini model. This would allow the model to access the tools via MCP.

( conversion of mcp_golang.ToolResponse into genai.Tool )

need to 

