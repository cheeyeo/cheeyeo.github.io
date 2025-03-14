---
layout: post
show_meta: true
title: Function Calling using Gemini API
header: Function Calling using Gemini API
date: 2025-03-13 00:00:00
summary: How to use function calling in Gemini API
categories: go-lang gemini function-calling agent
author: Chee Yeo
---

[Function Calling tutorial]: https://ai.google.dev/gemini-api/docs/function-calling/tutorial?lang=go 
[Open-Meteo go lang client]: https://github.com/hectormalot/omgo
[Gemini SDK Type]: https://pkg.go.dev/github.com/google/generative-ai-go/genai#Type

When working with foundational models, it not always possible to pass all the information or knowledge as context into a single prompt. Hence the need for architectures such as RAG and agentic workflows to be able to access external sources of information to augement the context of a query.

Most foundational models allow you to perform external actions through the use of `tools` which can be called through the model's API. The term `function calling` is used to describe the action of invoking a `tool` to perform an external action. Such actions could include calling an external API to retrieve information specific to a query such as a news API over the internet or to enhance a model's capability such as a code generator which could create and execute a piece of code and return its results.

In the context of Gemini API, Function Calling works by declaring a `function declaration` in a model prompt. A function declaration is a structured OpenAPI compatible schema that consists of:

* function name
* function description
* input parameters, each consisting of a type (string, array, number), an optional description and whether the parameter is required or not. 

For example, suppose we want to get the weather from a given location. We would need at least the following 2 components:
* A geocoder service to get our current latitude and longitude.
* A weather service which would accept the above coordinates and return the current temperature.

We could implement the above in go lang using the Google Maps service and the Open-Meteo weather API service. 

The following is the code for a possible geocoding service using Google Maps:

{% highlight go linenos %}
func getGeoCode(address string) (float64, float64, error) {
	gMapClient, err := maps.NewClient(maps.WithAPIKey(os.Getenv("GOOGLE_MAPS_API")))
	if err != nil {
		return float64(0), float64(0), err
	}

	r := &maps.GeocodingRequest{
		Address: address,
	}
	resp, err := gMapClient.Geocode(context.Background(), r)
	if err != nil {
		return float64(0), float64(0), err
	}

	return resp[0].Geometry.Location.Lat, resp[0].Geometry.Location.Lng, nil
}
{% endhighlight %}

The following is a possible implementation for the Open-Meteo service:

{% highlight go linenos %}
func getCurrentTemperature(lat float64, lng float64) (float64, error) {
	c, err := omgo.NewClient()
	if err != nil {
		return float64(0), err
	}

	loc, err := omgo.NewLocation(lat, lng)
	if err != nil {
		return float64(0), err
	}
	res, err := c.CurrentWeather(context.Background(), loc, nil)
	if err != nil {
		return float64(0), err
	}

	return res.Temperature, nil
}
{% endhighlight %}

Here we are using the [Open-Meteo go lang client] to retrieve the current weather based on the location parameters.

To use these functions in Gemini, we need to create **function declarations** of each of them. Using the Gemini SDK, we can create a `genai.FunctionDeclaration`. The function declaration needs to be set within a `genai.Tool` struct. In our example here, I decided to create each function as a separate tool. This also allows us to test if the model is capable of calling another function given the output of the first one.

The declaration for the getGeoCode function becomes:

{% highlight go linenos %}
{% raw %}
geocodeTool := &genai.Tool{
    FunctionDeclarations: []*genai.FunctionDeclaration{{
        Name:        "getGeoCode",
        Description: "Gets the latitude and longitude for a given address. Returns a location value of latitude and longitude.",
        Parameters: &genai.Schema{
            Type: genai.TypeObject,
            Properties: map[string]*genai.Schema{
                "address": {
                    Type:        1,
                    Description: "address to geocode",
                },
            },
            Required: []string{"address"},
        },
    }}
}
{% endraw %}
{% endhighlight %}

Note that we need to provide a name and description for the function declaration. The parameters field is optional. The function accepts a single parameter of `address` which we declare here to be of type String. Gemini SDK uses [Gemini SDK Type] to define the parameter type. This conforms to a subset of the OpenAPI specifications. We provide a short description for the parameter. Finally, we declare that the address parameter is required by listing it in the required list.

The function declaration for the getCurrentTemperature function is similar:

{% highlight go linenos %}
{% raw %}
temperatureTool := &genai.Tool{
    FunctionDeclarations: []*genai.FunctionDeclaration{{
        Name:        "getCurrentTemperature",
        Description: "Gets the current weather from the Open-Meteo API with given latitude and longitude parameters.",
        Parameters: &genai.Schema{
            Type: genai.TypeObject,
            Properties: map[string]*genai.Schema{
                "lat": {
                    Type:        2,
                    Description: "latitude of location",
                },
                "lng": {
                    Type:        2,
                    Description: "longitude of location",
                },
            },
            Required: []string{"lat", "lng"},
        },
    }},
}
{% endraw %}
{% endhighlight %}

Next, we create a model and add the above declarations into the model's tool inventory:

{% highlight go linenos %}
ctx := context.Background()
client, err := genai.NewClient(ctx, option.WithAPIKey(os.Getenv("API_KEY")))
if err != nil {
    log.Fatal(err)
}
defer client.Close()

// Use a model that supports function calling, like a Gemini 2.0 model
model := client.GenerativeModel("gemini-2.0-flash")
// Set the temperature to 0 to reduce hallucination
model.SetTemperature(0.0)

// Specify the function declaration.
model.Tools = []*genai.Tool{geocodeTool, temperatureTool}
{% endhighlight %}

Note that we created a Gemini 2.0 Flash model and set its **temperature** hyperparameter to 0. This helps to prevent the model from hallucinations and allows the model's responses to be more deterministic.

We create a chat session and sends a prompt which includes the address as a query for which we wish to obtain the temperature for:

{% highlight go linenos %}
// Start new chat session.
session := model.StartChat()

prompt := fmt.Sprintf(`
You are a helpful assistant that use tools to access and retrieve information from a weather API with a given address. 

Only use the following set of tools available to you:
- getGeoCode
- getCurrentTemperature

Use the given address to get the geolocation first. Use the geolocation to get the current temperature.

For each function call, show the parameters you use.

The address to get the weather for is %s.`, address)

// Send the message to the generative model.
resp, err := session.SendMessage(ctx, genai.Text(prompt))
if err != nil {
    log.Fatalf("Error sending message: %v\n", err)
}
{% endhighlight %}

Within the prompt, we explicity state the model's role and the tools it is able to use. We explicitly state the steps the model should take to get the current temperature. In an agentic workflow for example, we would be using an additional reasoning framework such as ReACT which would allow the model to plan and reason on the steps it should take. We also state that the model should show the parameters it use for each function call so we can visually inspect the model's responses. 

If function calling works, the first response we receive from the model would be a function call response, stating the name of the tool it will invoke and the parameters it's going to use. To validate this, we can cast the response part to type `genai.FunctionCall` and check that it matches with the function declarations made earlier:

{% highlight go linenos %}
// Check that you got the expected function call back.
part := resp.Candidates[0].Content.Parts[0]
funcall, ok := part.(genai.FunctionCall)
if !ok {
    log.Fatalf("Expected type FunctionCall, got %T", part)
}
if g, e := funcall.Name, geocodeTool.FunctionDeclarations[0].Name; g != e {
    log.Fatalf("Expected FunctionCall.Name %q, got %q", e, g)
}
fmt.Printf("Received function call response:\n%v\n\n", part)
{% endhighlight %}

If it's successful we should receive the following response:

{% highlight go linenos %}
Received function call response:
{getGeoCode map[address:Jurong Town, Singapore]}
{% endhighlight %}

The above shows that the model is able to invoke the function of `getGeoCode` with a single parameter of `address`.

Next, we need to return the above response as a result back to the model via a `genai.FunctionResponse`:

{% highlight go linenos %}
// Calls the getGeoCode function here
// In real world usage, we would have some validation / checks here...
lat, lng, err := getGeoCode(address)
if err != nil {
    log.Fatalf("Geocode err: %w", err)
}
log.Printf("LOCATION: %f %f\n", lat, lng)

apiResult := map[string]any{
    "lat": lat,
    "lng": lng,
}

fmt.Printf("Sending API result:\n%v\n\n", apiResult)
resp, err = session.SendMessage(ctx, genai.FunctionResponse{
    Name:     geocodeTool.FunctionDeclarations[0].Name,
    Response: apiResult,
})
if err != nil {
    log.Fatalf("Error sending message: %v\n", err)
}
{% endhighlight %}

In our example, we make an actual call to the function and set the results it returns as the actual response in the function response.

Next, we check that the model's response would consist of the second function call to getCurrentTemperature. We repeat the validation check on the second response:

{% highlight go linenos %}
// Show the model's response, which is to be the next expected function call
part = resp.Candidates[0].Content.Parts[0]
funcall, ok = part.(genai.FunctionCall)
if !ok {
    log.Fatalf("Expected type FunctionCall, got %T", part)
}
if g, e := funcall.Name, temperatureTool.FunctionDeclarations[0].Name; g != e {
    log.Fatalf("Expected FunctionCall.Name %q, got %q", e, g)
}
fmt.Printf("Received function call response:\n%q\n\n", part)

// Calls getCurrentTemperature here
currentTemp, err := getCurrentTemperature(lat, lng)
if err != nil {
    log.Fatalf("error with calling getCurrentTemperature - ", err)
}
apiResult = map[string]any{
    "temperature": currentTemp,
}

fmt.Printf("Sending 2nd API result:\n%v\n\n", apiResult)
resp, err = session.SendMessage(ctx, genai.FunctionResponse{
    Name:     temperatureTool.FunctionDeclarations[0].Name,
    Response: apiResult,
})
if err != nil {
    log.Fatalf("Error sending message: %v\n", err)
}

// The task would have ended here. The response should only be text
// Show the model's response, which is expected to be text.
for _, part := range resp.Candidates[0].Content.Parts {
    fmt.Printf("%v\n", part)
}
{% endhighlight %}

We call the actual getCurrentTemperature function and returns its return value as part of the `genai.FunctionResponse`. Now that both functions have been invoked, the model should recognise the end of the task. The final response should be a single text response which includes the current temperature.

Below is a terminal screenshot of the interaction with the model:
![Terminal output](/assets/img/gemini/functioncalling/screenshot.png)

Below is the full code listing:
<script src="https://gist.github.com/cheeyeo/9e21eb7f35ecad1f3b1f62a1680b5d0f.js"></script>
Note that you would require API Keys for Gemini and Google Maps to run the example.

To learn more about function calling, there is a [Function Calling tutorial] on the Gemini API documentation.

In summary, function calling and tools allow a model to inherit additional capabilities without needing to fine-tune the model. It also allows the model to interact with the external environment, in our example, the internet to retrieve specific information relevant to the query. 

H4PPY H4CK1NG!