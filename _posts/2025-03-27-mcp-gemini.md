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

[Model Context Protocol]: https://modelcontextprotocol.io/introduction
[MCP Golang]: https://github.com/metoro-io/mcp-golang/

In a previous blog post, I explained the use of Function Calling as a means of a foundational model interacting with the external world by calling pre-defined APIs with the required arguments which it learns through user prompts. Feedback loops such as reasoning and planning further enhances the model's ability to select the right tool to invoke given a particular set of circumstances.

Within Gemini API, to perform function calling, the documented approach is to define a list of tools using a `genai.Tool` struct. The main body of each tool struct contains the function calling declaration:

{% highlight go linenos %}
{% raw %}
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
{% endraw %}
{% endhighlight %}

While the above works, as the number of tools increase, we need to update the model code to register the new function calls. This means there is a 1-N relationship between the model code and the tools available. In addition, by coupling the function call code, we are unable to test the function calls separately. 

[Model Context Protocol] ( MCP ) is a protocol that standardizes how applications interact with foundational models. It provides a method to connect models to different data sources and resources. 

MCP has the following components:

* Hosts - models in use such as Gemini that need to retrieve the tools to use.

* MCP Client - Client that maintain a connection to the MCP Server. This is integrated with the model code as shown later.

* MCP Server - Application where the tools are registered.

* Local data sources - Files and services that the MCP servers can access.

* Remote Services - External services via APIs that the MCP servers can link to.

The following diagram below illustrates the above definitions. Note that, we do not need to declare all the function declarations on 1 server. We can break it up into different MCP servers depending on the architecture of the system.

![MCP architecture](/assets/img/mcp/architecture.png)

For this example, we are using the [MCP Golang] library.

For each tool to perform function call on, we register it within the MCP server. Each definition  has a name, description and a function handler that accepts the arguments passed in and returns a suitable response via the `mcp_golang.ToolResponse` type:

{% highlight go linenos %}
{% raw %}
err := server.RegisterTool("hello", "Say hello to a person", func(args HelloArgs) (*mcp_golang.ToolResponse, error) {
		message := fmt.Sprintf("Hello %s!", args.Name)
		return mcp_golang.NewToolResponse(mcp_golang.NewTextContent(message)), nil
	})
{% endraw %}
{% endhighlight %}

The example above registers a tool called `hello` with a single argument of `Name`. To call it using the MCP client:

{% highlight go linenos %}
{% raw %}
helloArgs := map[string]interface{}{
    "name": "World!",
}

helloResp, err := client.CallTool(context.Background(), "hello", helloArgs)

if err != nil {
    log.Printf("Failed to call hello tool: %v", err)
} else {
    log.Printf("Hello response: %+v", helloResp.Content[0].TextContent.Text)
}
{% endraw %}
{% endhighlight %}


### Connecting MCP to Gemini

To connect MCP to Gemini, we need the following:

* To register each tool on MCP server.
* Pass a list of tools to the Gemini model via `client.ListTools`
* Convert each tool from jsonschema to OpenAPI schema in Gemini.
* Create and pass prompt to Gemini.
* Get response from Gemini.

[this article]: https://www.bytesizego.com/blog/model-context-protocol-golang
For this post, I'm going to reuse an example from [this article] which has an interesting example in GoLang that calls out to an external 3rd party API to retrieve bitcoin value based on the passed in currency value.

For the first step, we initialize the server and call `server.RegisterTool`:

{% highlight go linenos %}
{% raw %}
...

type BitcoinPriceArguments struct {
	Currency string `json:"currency" jsonschema:"required,description=The currency to get the Bitcoin price in (USD, EUR, GBP, etc)"`
}

// Register the bitcoin_price tool
err = server.RegisterTool("bitcoin_price", "Get the latest Bitcoin price in various currencies", func(arguments BitcoinPriceArguments) (*mcp_golang.ToolResponse, error) {
    log.Printf("received request for bitcoin_price tool with currency: %s", arguments.Currency)

    currency := arguments.Currency
    if currency == "" {
        currency = "USD"
    }

    // Call CoinGecko API to get latest Bitcoin price
    price, err := getBitcoinPrice(currency)
    if err != nil {
        return mcp_golang.NewToolResponse(mcp_golang.NewTextContent(fmt.Sprintf("Error fetching Bitcoin price: %v", err))), err
    }

    return mcp_golang.NewToolResponse(mcp_golang.NewTextContent(fmt.Sprintf("The current Bitcoin price in %s is %.2f (as of %s)",
        currency,
        price,
        time.Now().Format(time.RFC1123)))), nil
})
if err != nil {
    log.Fatalf("error registering bitcoin_price tool: %v", err)
}
{% endraw %}
{% endhighlight %}

The above registers a tool of name `bitcoin_price` which we can perform function calling on by passing arguments of type `BitCoinArguments`. The currency value is extracted and calls `getBitcoinPrice` which is an external API that returns a list of bitcoin prices. The external API response is wrapped in `mcp_golang.NewToolResponse` which returns a string of the values.

The next step is to pass the list of tools available from MCP server to the Gemini model. The MCP client can all `client.ListTools` which returns a list of the tools available. We traverse this list of tools and convert each into a `genai.Tool` type:

{% highlight go linenos %}
{% raw %}
// List available tools on MCP server
	tools, err := client.ListTools(context.Background(), nil)
	if err != nil {
		log.Fatalf("Failed to list tools: %v\n", err)
	}

	log.Println("Available tools:")
	// Create list of gemini tools
	geminiTools := []*genai.Tool{}

	for _, tool := range tools.Tools {
		desc := ""
		if tool.Description != nil {
			desc = *tool.Description
		}
		log.Printf("Tool: %s. Description: %s, Schema: %+v", tool.Name, desc, tool.InputSchema)

		// Cast inputschema from interface to map[string]any
		inputDict := tool.InputSchema.(map[string]any)
		jsonbody, err := json.Marshal(inputDict)
		if err != nil {
			log.Fatalf("error with converting tool.InputSchema - %s", err)
		}

		gschema := GSchema{}
		err = json.Unmarshal(jsonbody, &gschema)
		if err != nil {
			log.Fatalf("error with converting tool.InputSchema - %s", err)
		}

		geminiProperties, err := gschema.Convert()
		geminiTool := &genai.Tool{
			FunctionDeclarations: []*genai.FunctionDeclaration{{
				Name:        tool.Name,
				Description: *tool.Description,
				Parameters:  geminiProperties,
			}},
		}
		geminiTools = append(geminiTools, geminiTool)
	}
{% endraw %}
{% endhighlight %}

Each tool response is of `mcp_golang.ToolRetType`. It has the following structure:

{% highlight go linenos %}
{% raw %}
type ToolRetType struct {
	// A human-readable description of the tool.
	Description *string `json:"description,omitempty" yaml:"description,omitempty" mapstructure:"description,omitempty"`

	// A JSON Schema object defining the expected parameters for the tool.
	InputSchema interface{} `json:"inputSchema" yaml:"inputSchema" mapstructure:"inputSchema"`

	// The name of the tool.
	Name string `json:"name" yaml:"name" mapstructure:"name"`
}
{% endraw %}
{% endhighlight %}

A `genai.Tool` struct has the following format:

{% highlight go linenos %}
{% raw %}
type FunctionDeclaration struct {
	// Required. The name of the function.
	// Must be a-z, A-Z, 0-9, or contain underscores and dashes, with a maximum
	// length of 63.
	Name string
	// Required. A brief description of the function.
	Description string
	// Optional. Describes the parameters to this function. Reflects the Open
	// API 3.03 Parameter Object string Key: the name of the parameter. Parameter
	// names are case sensitive. Schema Value: the Schema defining the type used
	// for the parameter.
	Parameters *Schema
}
{% endraw %}
{% endhighlight %}

From above, we can see that the `genai.FunctionDeclaration.Parameters` field has a structure that conforms to the Open API 3.0 spec whereas the `mcp_golang.ToolRetType.InputSchema` is of spec JSON schema. We need to convert the `InputSchema` type to `genai.Schema` type.

The first approach I took was to cast it to `map[string]any`. This would allow for JSON serialization. We create a custom struct that contains the same fields as `InputSchema`. Then, we create a function `Convert()` on the struct type to perform the conversion to genai.Schema for the tool's function declaration parameters:

{% highlight go linenos %}
{% raw %}
type Property struct {
	Description string `json:"description"`
	Type        string `json:"type"`
}

type GSchema struct {
	Schema     string              `json:"$schema"`
	Properties map[string]Property `json:"properties"`
	Required   []string            `json:"required"`
	Type       string              `json:"type"`
}

func getType(kind string) (genai.Type, error) {
	var gType genai.Type
	switch kind {
	case "object":
		gType = genai.TypeObject
	case "array":
		gType = genai.TypeArray
	case "string":
		gType = genai.TypeString
	case "number":
		gType = genai.TypeNumber
	case "integer":
		gType = genai.TypeInteger
	case "boolean":
		gType = genai.TypeBoolean
	default:
		return 0, fmt.Errorf("type not found in gemini Type: %s", kind)
	}

	return gType, nil
}

func (g GSchema) Convert() (*genai.Schema, error) {
	var parseErr error
	res := &genai.Schema{}

	gType, parseErr := getType(g.Type)
	if parseErr != nil {
		return nil, parseErr
	}
	res.Type = gType
	res.Required = g.Required

	// Convert properties to map of genai.Schema
	schemaProperties := map[string]*genai.Schema{}
	for k, v := range g.Properties {
		gType, parseErr := getType(v.Type)
		if parseErr != nil {
			return nil, parseErr
		}
		schemaProperties[k] = &genai.Schema{
			Description: v.Description,
			Type:        gType,
		}
	}
	res.Properties = schemaProperties

	return res, nil
}

func main() {
    ...

    // Cast inputschema from interface to map[string]any
    inputDict := tool.InputSchema.(map[string]any)
    jsonbody, err := json.Marshal(inputDict)
    if err != nil {
        log.Fatalf("error with converting tool.InputSchema - %s", err)
    }

    gschema := GSchema{}
    err = json.Unmarshal(jsonbody, &gschema)
    if err != nil {
        log.Fatalf("error with converting tool.InputSchema - %s", err)
    }

    geminiProperties, err := gschema.Convert()

    geminiTool := &genai.Tool{
			FunctionDeclarations: []*genai.FunctionDeclaration{{
				Name:        tool.Name,
				Description: *tool.Description,
				Parameters:  geminiProperties,
			}},
		}
	
    geminiTools = append(geminiTools, geminiTool)
}
{% endraw %}
{% endhighlight %}

Finally, we can initialize the Gemini model and pass the list of tools to it. We pass the user prompt to the model and parse its response. If the function call is successful, we pass the same function name and args to the MCP server via the MCP client and parse the response. This response is passed back to the Gemini model as a `FunctionResponse` struct:

{% highlight go linenos %}
{% raw %}
model := geminiClient.GenerativeModel("gemini-2.5-pro-preview-03-25")
model.Tools = geminiTools
model.SetTemperature(0.1)

session := model.StartChat()

prompt := "What's the current Bitcoin price in GBP?"

res, err := session.SendMessage(ctx, genai.Text(prompt))
if err != nil {
    log.Fatalf("session.SendMessage: %v", err)
}

part := res.Candidates[0].Content.Parts[0]
funcall, ok := part.(genai.FunctionCall)
log.Printf("gemini funcall: %+v\n", funcall)
if !ok {
    log.Fatalf("expected functioncall but received error:\n%v", part)
}

// Make actual call in MCP
var geminiFunctionResponse map[string]any
helloResp, err := client.CallTool(context.Background(), funcall.Name, funcall.Args)
if err != nil {
    log.Printf("failed to call tool: %v\n", err)
    geminiFunctionResponse = map[string]any{"error": err}
} else {
    log.Printf("Response: %v\n", helloResp.Content[0].TextContent.Text)
    geminiFunctionResponse = map[string]any{"response": helloResp.Content[0].TextContent.Text}
}

// Send resp back to gemini
res, err = session.SendMessage(ctx, genai.FunctionResponse{
    Name:     funcall.Name,
    Response: geminiFunctionResponse,
})
if err != nil {
    log.Fatal(err)
}
{% endraw %}
{% endhighlight %}

The following is the terminal response from running the above:

{% highlight shell linenos %}
2025/03/27 18:11:31 Available tools:
2025/03/27 18:11:31 Tool: bitcoin_price. Description: Get the latest Bitcoin price in various currencies, Schema: map[$schema:https://json-schema.org/draft/2020-12/schema properties:map[currency:map[description:The currency to get the Bitcoin price in (USD type:string]] required:[currency] type:object]
2025/03/27 18:11:31 Tool: hello. Description: Say hello to a person, Schema: map[$schema:https://json-schema.org/draft/2020-12/schema properties:map[name:map[description:The name to say hello to type:string]] required:[name] type:object]

2025/03/27 18:11:33 gemini funcall: {Name:bitcoin_price Args:map[currency:GBP]}
2025/03/27 18:11:34 MCP Response: The current Bitcoin price in GBP is 63579.00 (as of Thurs, 27 Mar 2025 18:11:34 BST)

The current Bitcoin price in GBP is £63,579.00 (as of Thurs, 27 Mar 2025 18:11:34 BST).
---
{% endhighlight %}

Note that the Gemini model is able to recognise that it needs to perform a function call from the prompt asking for the current price of bitcoin in GBP. It returns a response struct of function name `bitcoin_price` and the currency argument to `GBP`. This is invoked by the MCP client which passes it to the MCP server. The model receives the response and returns the final text response, which matches the MCP server response.

Another advantage of using MCP is error handling. For the given example above, if the wrong currency value is passed to the tool, the model is able to return an error response. For example, if the currency value is `test` we get the following response back:

{% highlight shell linenos %}
expected functioncall but received error:

I cannot get the Bitcoin price in "test" as it's not a recognized currency. Please provide a standard currency code like USD, EUR, GBP, etc.
{% endhighlight %}

The full gist can be found below:

<script src="https://gist.github.com/cheeyeo/d3a5a500e2160b4d777f81146648249f.js"></script>