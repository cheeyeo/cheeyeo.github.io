---
layout: post
show_meta: true
title: Convert nested form object to compatible OpenAPI schema for Gemini
header: Convert nested form object to compatible OpenAPI schema for Gemini
date: 2025-02-18 00:00:00
summary: How to convert ested object form for an openapi compatible schema
categories: javascript nested-form openapi schema go-lang gemini
author: Chee Yeo
---

[previous post on nested objects form]: https://cheeyeo.xyz/javascript/nested-form/2025/02/01/nested-javascript-form/
[Generate structured output]: https://ai.google.dev/gemini-api/docs/structured-output?lang=go
[Gemini API Schema]: https://ai.google.dev/api/caching#Schema
[OpenAPI 3.0 Schema Object]: https://spec.openapis.org/oas/v3.0.3#schema-object

In [previous post on nested objects form], I explained an approach of using recursion to iterate over nested objects created dynamically via javascript in a given form. Given an example object of a recipe, its JSON output has the following structure:

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

Each parameter from the form is mapped into a param object. The `type` field indicates the category of the parameter kind. In this example, it's a type of object denoted by type 6, with an attribute of `name` of type string denoted by 1.

The Gemini API allows a user to [Generate structured output]. By default, Gemini returns free-form text. We can constrain the model to only return the output formatted in JSON with specific attributes and values that we specify. This can be achieved by:

* Specifying the structure of the output directly in the prompt.

* Create a response schema and pass that into the model's generate content call.

By creating a response schema, we have more control and predictability of the model's outputs. However, the specification states that the Gemini's response schema uses a subset of the OpenAPI 3.0 Schema Object. After some experimentation, I managed to create an approach to remap the JSON output into the required schema format.

The following is an example of the [Gemini API Schema]. Note that this is a subset of [OpenAPI 3.0 Schema Object] that the Gemini API accepts:

{% highlight json linenos %}
{
  "type": enum (Type),
  "format": string,
  "description": string,
  "nullable": boolean,
  "enum": [
    string
  ],
  "maxItems": string,
  "minItems": string,
  "properties": {
    string: {
      object (Schema)
    },
    ...
  },
  "required": [
    string
  ],
  "propertyOrdering": [
    string
  ],
  "items": {
    object (Schema)
  }
}
{% endhighlight %}

The initial recipe example translated into the schema would look like this:
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
            "required":[]
        }
    },
    "required":[]
}
{% endhighlight %}

An example of an array of recipes becomes:
{% highlight json linenos %}
{
    "type":6,
    "properties":{
        "recipe":{
            "type":5,
            "items":{
                "type":6,
                "properties":{
                    "name":{
                        "type":1
                    }
                },
                "required":[]
            }
        }
    },
    "required":[]
}
{% endhighlight %}

Note that we have 2 cases whereby we can have a self-referenced schema object:
* within an array
* nested objects within objects

My solution is to create a separate function that would remap the form output from before. The function would traverse each parameter in the form object and translate it into the OpenAPI schema equivalent:

{% highlight javscript linenos %}
formObj = {
    "params":[
        {
            "name":"recipe",
            "type":"6",
            "objects":[{"name":"name","type":"1","items":"TRUE"}]
        }
    ]
}

 // Convert to openapi schema format
var customSchema = {}
customSchema.type = 6;
customSchema.properties = {};
customSchema.required = [];

for (let[key, value] of Object.entries(formObj.params)) {
    customSchema.properties[value.name] = {};

    switch (value.items) {
        case "TRUE":
            customSchema.properties[value.name].type = 5;

            if (value.type === "6") {
                customSchema.properties[value.name].items = {
                    "type": 6,
                    "properties": {},
                    "required": []
                };

                parseObjs(value.objects, customSchema.properties[value.name].items.properties, customSchema.properties[value.name].items.required)

                // set required if exists
                if (value.required) {
                    customSchema.required.push(value.name)
                }
            } else if (value.type === "enum") {
                let tmpEnum = [];
                for (let[id, enumVal] of Object.entries(value.enum)) {
                    tmpEnum.push(enumVal)
                }
                
                customSchema.properties[value.name].items = {
                    "type": 1,
                    "enum": tmpEnum
                }
                // set required if exists
                if (value.required) {
                    customSchema.required.push(value.name)
                }
            } else {
                customSchema.properties[value.name].items = {
                    "type": parseInt(value.type)
                }
                // set required if exists
                if (value.required) {
                    customSchema.required.push(value.name)
                }
            }
            
            break;
        default:
          // omitted for brevity
    }
}
{% endhighlight %}

We define a schema object as an object with a attribute type of 6 (corresponds to object type in Gemini schema) and initialises its `properties` and `required` fields.

The function iterates over every parameter. Within the loop we initialize the parameter's name as an object in the schema properties. Next, it checks the parameter type via the switch block. The first case statement checks if the parameter is an array by comparing its `.items` attribute. If its true, we make the following checks:

* if the item is an object type; if it is, we need to check if it has nested objects by passing it to the `parseObjs` function
* if its an enum type; if it is, we flatten the enum array into a single array and assign it the type of string ( corresponds to 1 in Gemini schema )
* if its a scalar type ( integer, string ); if it is we directly assign it to the properties object with the parameter name and type.

{% highlight javscript linenos %}
function parseObjs(objs, target, targetReq) {
    for (let[key, value] of Object.entries(objs)) {
        target[value.name] = {"type": parseInt(value.type)}
        if (value.required) {
            targetReq.push(value.name)
        }

        switch (value.items) {
            case "TRUE":
                target[value.name].type = 5;

                if (value.type === "6") {
                    target[value.name].items = {
                        "type": 6,
                        "properties": {},
                        "required": []
                    };

                    parseObjs(value.objects, target[value.name].items.properties, target[value.name].items.required)
                } else if (value.type === "enum") {
                    let tmpEnum = [];
                    for (let[id, enumVal] of Object.entries(value.enum)) {
                        tmpEnum.push(enumVal)
                    }
                    
                    target[value.name].items = {
                        "type": 1,
                        "enum": tmpEnum
                    }
                } else {
                    target[value.name].items = {
                        "type": parseInt(value.type)
                    }
                }
                
                break;
            default:
              // omitted for brevity
        }
    }
}
{% endhighlight %}

The `parseObjs` function is a recursive function that takes as inputs a list of parameters; a target object and a list of required parameters. We iterate over the input parameters and check if the parameter is a list; every other type is delegated to the default block, using the same sequence of logic as from the conversion function. If the parameter is a list, we check:

* if the item is an object type; if it is, we need to check if it has nested objects by passing it to the `parseObjs` function, making a recursive call.
* if its an enum type; if it is, we flatten the enum array into a single array and assign it the type of string ( corresponds to 1 in Gemini schema )
* if its a scalar type ( integer, string ); if it is we directly assign it to the properties object with the parameter name and type.

I deliberately left out the `default` code blocks in both cases as they follow the exact same logic in the array case, with changes to the target properties object to write to.

To test the model output, I used the go-lang example from the documentation with some changes to creating a custom struct to serialize the generated schema json into the `genai.Schema` type:

{% highlight go linenos %}
func main() {
    // input to test
	jsonInput := ``

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

	resp, err := model.GenerateContent(ctx, genai.Text("List 10 cookie recipes using this JSON schema."))

	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("resp: %+v\n", resp.Candidates[0].Content.Parts[0])
}
{% endhighlight %}

Given the initial single recipe example of type object with a single name attribute, we obtain a JSON schema of:

![Nested form](/assets/img/nested-form-2/screenshot1.png)

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
            "required":[]
        }
    },
    "required":[]
}
{% endhighlight %}

The output from the Gemini structured output call:

{% highlight json linenos %}
{
    "recipe": {
        "name": 
        "Chocolate Chip Cookies"
    }
}
{% endhighlight %}

An example of nested recipe objects with the ingredients as a nested list of type object within each recipe object:

{% highlight json linenos %}
{
    "type":6,
    "properties":{
        "recipe":{
            "type":5,
            "items":{
                "type":6,
                "properties":{
                    "name":{
                        "type":1
                    },
                    "ingredients":{
                        "type":5,
                        "items":{
                            "type":6,
                            "properties":{
                                "name":{
                                    "type":1
                                },
                                "qty":{
                                    "type":1
                                },
                                "unit":{
                                    "type":1
                                }
                            },
                            "required":[]
                        }
                    }
                },
                "required":[]
            }
        }
    },
    "required":[]
}
{% endhighlight %}

Gemini response:
{% highlight json linenos %}
{
    "recipe": [
        {
            "ingredients": [
                {"name": "All-purpose flour", "qty": "3", "unit": "cups"}, 
                {"name": "Baking soda", "qty": "1", "unit": "teaspoon"}, 
                {"name": "Salt", "qty": "1/2", "unit": "teaspoon"}, 
                {"name": "Unsalted Butter", "qty": "1", "unit": "cup"}, 
                {"name": "Granulated sugar", "qty": "1", "unit": "cup"}, 
                {"name": "Brown sugar", "qty": "1", "unit": "cup"}, 
                {"name": "Large Eggs", "qty": "2"}, 
                {"name": "Vanilla extract", "qty": "1", "unit": "teaspoon"}, 
                {"name": "Chocolate chips", "qty": "2", "unit": "cups"}
            ], 
            "name": "Chocolate Chip Cookies"
        }, 
        {
            "ingredients": [
                {"name": "All-purpose flour", "qty": "2 1/2", "unit": "cups"}, 
                {"name": "Baking powder", "qty": "1", "unit": "teaspoon"}, 
                {"name": "Salt", "qty": "1/4", "unit": "teaspoon"}, 
                {"name": "Unsalted butter", "qty": "1", "unit": "cup"}, 
                {"name": "Granulated sugar", "qty": "1", "unit": "cup"}, 
                {"name": "Egg", "qty": "1"}, 
                {"name": "Vanilla extract", "qty": "1/2", "unit": "teaspoon"}, 
                {"name": "Sprinkles", "qty": "1/2", "unit": "cup"}
            ], 
            "name": "Sugar Cookies"
        },

        ...
    ]
}
{% endhighlight %}

With the above approach, I was able to automate the process of generating response schemas from the UI to be passed as part of a structured output call to Gemini.

In the next article, I will highlight how I used the processed schema JSON and convert it into a valid genai.Schema struct.

H4PPY H4CK1NG