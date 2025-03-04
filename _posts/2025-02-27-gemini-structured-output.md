---
layout: post
show_meta: true
title: Gemini structured output call from UI
header: Gemini structured output call from UI
date: 2025-02-27 00:00:00
summary: How to call Gemini structured output from UI
categories: openapi schema go-lang gemini structured-output
author: Chee Yeo
---

[previous post on dynamically creating Gemini Schema via javascript]: https://cheeyeo.dev/javascript/nested-form/openapi/schema/go-lang/gemini/2025/02/18/openapi-schema-form/

In my [previous post on dynamically creating Gemini Schema via javascript], I discussed how I used javascript to convert a nested object form first to JSON and then remapping the JSON object to a Gemini compatible Response schema.

For example, given the following JSON object created via nested forms:

{% highlight json linenos %}
{
    "params":[
        {
            "name":"recipe",
            "type":"6",
            "objects":[
                {
                    "name":"name",
                    "type":"1"
                }
            ]
        }
    ]
}
{% endhighlight %}

its converted to:

{% highlight json linenos %}
{
    "type":6,
    "properties":{
        "recipe":{
            "type":6,
            "properties":{
                "name":{
                    "type":1
                }
            },
        }
    },
}
{% endhighlight %}

The above is passed as input to the `go-lang` script which then invokes `GenerateContent` to create a JSON response of the prompt input. The example I'm using below is borrowed from the official documentation:

{% highlight go linenos %}
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"

	"github.com/google/generative-ai-go/genai"
	"google.golang.org/api/option"
)

type CustomSchema struct {
	Type        genai.Type               `json:"type"`
	Format      string                   `json:"format"`
	Description string                   `json:"description"`
	Nullable    bool                     `json:"nullable"`
	Enum        []string                 `json:"enum"`
	Items       *genai.Schema            `json:"items"`
	Properties  map[string]*genai.Schema `json:"properties"`
	Required    []string                 `json:"required"`
}

func main() {
	jsonInput := `
{
    "type":6,
    "properties":{
        "recipe":{
            "type":6,
            "properties":{
                "name":{
                    "type":1
                }
            },
        }
    },
}`

	var mySchema CustomSchema
	err := json.Unmarshal([]byte(jsonInput), &mySchema)
	if err != nil {
		panic(err)
	}

	ctx := context.Background()
	client, err := genai.NewClient(ctx, option.WithAPIKey(os.Getenv("API_KEY")))
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()

	model := client.GenerativeModel("gemini-1.5-pro-latest")
	// Ask the model to respond with JSON.
	model.ResponseMIMEType = "application/json"

	// convert from customschema to genai.Schema
	target := &genai.Schema{}
	temp, _ := json.Marshal(mySchema)
	err = json.Unmarshal(temp, target)
	if err != nil {
		panic(err)
	}
	model.ResponseSchema = target

	resp, err := model.GenerateContent(ctx, genai.Text("List cookie recipes using this JSON schema."))

	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("RESP: %+v\n", resp.Candidates[0].Content.Parts[0])
}

{% endhighlight %}

In order to deserialize the json input into `genai.Schema` type, we need to define a custom type in order to add the json tags to the struct. My initial approach was to parse the input json and try to create the schema struct dynamically. However, the resulting code was dense and unreadable. Here, I'm directly delegating the deserialization to the `json` library. 

To handle nested objects in the JSON, one can self-reference struct field types to itself. For example, given that arrays and objects can also contain other objects, the original `genai.Schema` struct has this recursive self-reference as follows:

{% highlight go linenos %}
type Schema struct {
	Type        genai.Type               `json:"type"`
	Format      string                   `json:"format"`
	Description string                   `json:"description"`
	Nullable    bool                     `json:"nullable"`
	Enum        []string                 `json:"enum"`
	Items       *genai.Schema            `json:"items"`
	Properties  map[string]*genai.Schema `json:"properties"`
	Required    []string                 `json:"required"`
}
{% endhighlight %}

As mentioned previously, since `Items` denote an array, we create a self-reference pointer as it can contain other objects. Likewise, `Properties` is declared as a map of `genai.Schema` struct since it can also contain other objects.

In the example above, I defined a struct of CustomSchema which is exactly the same as `genai.Schema` but with added json tags to match the attribute names of the json input parameter.

We initially deserialize the json input into `mySchema` which is of type `CustomSchema`. We initialize a target type of `genai.Schema`. We serialize the `mySchema` created before by calling `json.Marshal(mySchema)` into a temporary variable. Finally, we deserialize the temporary variable into the target schema variable. This approach allows one to convert from one type of struct to another, provided they have the same struct field names.

By using this approach, I was able to pass the generated JSON from the UI directly to the backend code in an automated fashion.

H4PPY H4CK1NG