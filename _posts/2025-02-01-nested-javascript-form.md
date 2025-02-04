---
layout: post
show_meta: true
title: Nested form in javascript
header: Nested form in javascript
date: 2025-02-01 00:00:00
summary: How to access create a nested form in pure javascript
categories: javascript nested-form
author: Chee Yeo
---

[form-to-object]: https://github.com/serbanghita/form-to-object
[Function Calling in Gemini]: https://ai.google.dev/gemini-api/docs/function-calling/tutorial?lang=go
[Structured Output in Gemini]: https://ai.google.dev/gemini-api/docs/structured-output?lang=go

In a recent personal web project, I tried to implement a deeply nested form field. Previously, when using web frameworks such as Ruby-On-Rails, this is built into the framework through helpers such as `nested_form_fields`, which hides most of the implementation details. 

Since I'm building my entire application manually from scratch, I tried to implement it from scratch using pure javascript. The main rationale for my choice was that I didn't want to introduce another framework such as ReactJs when the current UI and javascript is already established. 

![Nested form](/assets/img/nested-form/form1.png)

Above is a screenshot of the form in question. It allows you to dynamically create a genai data schema which is passed downstream when doing [Function Calling in Gemini] or [Structured Output in Gemini].

Each schema consists of a name, description and a list of parameters. Each parameter has a name, type. It has attributes which marks it as either an array or required. The parameter type can be a scalar value such as a string or number but it can also be a complex type such as an object or an enumeration ( enum ) which means it needs to be able to store multiple values or in the case of an object, it can also contain other scalar attributes or even nested objects.

The example screenshot shows an example of creating a schema for a function call `controlLight` which takes 2 parameters: a number called brightness and a string called colour. The data schema created has the following structure:
{% highlight javascript linenos %}
{
    "name":"controlLight",
    "description":"Set the brightness and color temperature of a room light.","params":[
      {"name":"brightness","type":"2"},
      {"name":"colour","type":"1"}
    ],
    "required":["brightness","colour"]
}
{% endhighlight %}

The `Add Params` button has an event handler which calls out to a function `addParams` which creates a nested form field dynamically:

{% highlight javascript linenos %}
document.getElementById("add-params").addEventListener("click", addParams);

function addParams(event) {
    event.preventDefault();

    inline = `
    <div class="col-md-4">
        <label class="form-label">Param Name</label>
        <input type="text" class="form-control" id="params" name="params[${x}][name]">
    </div>
    <div class="col-md-4">
        <label class="form-label">Param Type</label>
        <select class="form-select" id="params-type" name="params[${x}][type]" data-path="params[${x}]">
            <option value="1">string</option>
            <option value="2">number</option>
            <option value="3">integer</option>
            <option value="4">boolean</option>
            <option value="6">object</option>
            <option value="enum">enum</option>
        </select>
    </div>
    <div class="col-md-4">
        <label class="form-label">Options</label>
        <div class="form-control">
            <a href="#" id="params-array">Array</a>
            <a href="#" id="params-required-link">Required</a>
            <a href="#" class="icon-link" id="delete-params"><i class="bi bi-trash"></i> Delete</a>

            <input type="hidden" id="params-items" name="params[${x}][items]" value="" />
            <input type="hidden" id="params-required" name="required[${x}]" value=""/>
        </div>
    </div>
    `

    var item = document.createElement("div");
    item.className = "row mb-3";
    item.id = `param-${x}`;
    item.setAttribute("data-id", x)
    item.innerHTML = inline;

    item.querySelector("a#params-array").addEventListener("click", setArray);
    item.querySelector("a#params-required-link").addEventListener("click", setRequired);
    item.querySelector("a#delete-params").addEventListener("click", deleteParams);
    item.querySelector("select#params-type").addEventListener("change", setType);
    document.getElementById("params").appendChild(item);
    x++;
}
{% endhighlight %}

Note that we use a variable `x` to keep track of each parameter created. I used an external plugin [form-to-object] to parse the form inputs. We need to subscript each parameter field with an index such as `params[${x}]` in order to create a object. For instance, `params[0][name]` refers to the first parameter name attribute. We use two hidden form fields to keep track of any array or required items.

Depending on the parameter type, we need to render a different form structure. For example, when creating a enumeration type, we provide the option of allowing the user to specify individual values via `Add Enum Value` as show below:

![Nested form with enum](/assets/img/nested-form/form2.png)

This is handled via another event handler:
{% highlight javascript linenos %}
function addEnum(event) {
    event.preventDefault();
    var id = event.target.parentNode.dataset.id;
    let path = event.target.getAttribute("data-path");

    var inline = `
        <div class="col-md-6 mb-3">
            <input type="text" class="form-control" id="params" name="${path}[enum][]" />
        </div>
        <div class="col-md-2 mt-2" id="delete-enum">
            <a href="#" id="delete-enum">Delete Enum</a>
        </div>
    `
    var item = document.createElement("div");
    item.className = "row px-4";
    item.id = "param";
    item.innerHTML = inline;
    item.querySelector("a#delete-enum").addEventListener("click", deleteEnum);
    event.target.parentNode.querySelector("#params-enum").append(item);
}
{% endhighlight %}

To keep track of which parameter this enumeration belongs to, we add a data attribute to the `Add Enum Value` link of the form:

{% highlight html %}
<a class="icon-link px-5" id="add-enum" href="#" data-path="params[0]">Add Enum Value</a>
{% endhighlight %}

The `data-path` indicates that this enum is to be created for `params[0]`. This is interpolated into the enum input field:
{% highlight html %}
<input type="text" class="form-control" id="params" name="params[0][enum][]">
{% endhighlight %}

Note that the enumeration is an array and so we need to further subscript it by adding square brackets to the end. The following schema is an example of a parameter with an enumeration:
{% highlight json linenos %}
{
    "params":[
      {
        "name":"color",
        "type":"enum",
        "enum":["dim","warm","daylight"]
      }
    ]
}
{% endhighlight %}

Suppose we need to create a schema where one of the attributes is an object. The following screenshot uses an example to create a schema for recipes where each recipe is an object with several attributes:

![Nested form with enum](/assets/img/nested-form/form_recipes.png)

{% highlight json linenos %}
{
    "name":"recipes",
    "params":[
      {
        "name":"recipe",
        "type":"6",
        "objects":[
          {"name":"recipeName","type":"1"},
          {"name":"desc","type":"1"},
          {"name":"ingredients","type":"1"},
          {"name":"instructions","type":"1"}
        ]
      }
    ]
}
{% endhighlight %}

Given the example above, the input field name has the following format:
{% highlight html %}
<input type="text" class="form-control" id="params-name" name="params[0][objects][0][name]">
{% endhighlight %}

Note that the name has two further subscripts: `objects[0][name]`. This means that the input belongs to the first object with a value of `name`.

Objects can also be other objects. The screenshot below shows a nested object which is 3 levels deep:

![Nested form](/assets/img/nested-form/form3.png)

The schema produced is:
{% highlight json linenos %}
{
    "name":"nestedObject",
    "params":[{
        "name":"OBJ1",
        "type":"6",
        "objects":[{
            "name":"OBJ2",
            "type":"6",
            "objects":[{
                "name":"OBJ3",
                "type":"6",
                "objects":[{
                    "name":"NAME",
                    "type":"1"
                }]
            }]
        }]
    }]
}
{% endhighlight %}

The input field for the name attribute becomes:
{% highlight html %}
<input type="text" class="form-control" id="params-name" name="params[0][objects][0][objects][0][objects][0][name]">
{% endhighlight %}

This is made possible by adding a `data-path` attribute as shown earlier in the enumeration exaample:
{% highlight html %}
<a class="icon-link px-5" id="add-objs" href="#" data-path="params[0][objects][0][objects][0]">Add Obj Property</a>
{% endhighlight %}

The complexity comes when parsing the form inputs to render it into JSON for submission to the backend. How does one parse the nested object structure? The [form-to-object] plugin was able to parse the nested form but it returns each nested object or enum value as a nested object:

{% highlight json linenos %}
{
    "name":"nestedObject",
    "params":{
        "0":{
            "name":"OBJ1",
            "type":"6",
            "objects":{
                "0":{
                    "name":"OBJ2",
                    "type":"6",
                    "objects":{
                        "0":{
                            "name":"OBJ3",
                            "type":"6",
                            "objects":{
                                "0":{"name":"NAME","type":"1"}
                            }
                        }
                    }
                }
            }
        }
    }
}
{% endhighlight %}

Rather than further conversions on the backend, I decided to create my own custom parsing function after the plugin has parsed the form:

{% highlight javascript linenos %}
function convertObjects(params) {
    for (let[key, value] of Object.entries(params)) {
        if (value.type === "6") {
            console.log("FOUND NESTED OBJ!!!!")
            // value of form {name: "XXX", type: "6", objects: {}}
            if (value.objects) {
                tmpArray = []
                for (let[key2, obj2] of Object.entries(value.objects)) {
                    // If enum present we need to flatten it
                    if (obj2.enum) {
                        let tmpEnum = []
                        for (let[id, enumVal] of Object.entries(obj2.enum)) {
                            tmpEnum.push(enumVal)
                        }
                        obj2.enum = tmpEnum
                    }
                    
                    tmpArray[key2] = obj2
                }
                value.objects = tmpArray
            }
            // Check for nested objects
            convertObjects(value.objects)
        }
    }
}
{% endhighlight %}

The function loops through each of the params attributes returned by the form plugin. If its of type object, it loops through each of its attributes and converts it into an array. To further check if the object contains another nested object, it performs recursion by calling itself again and further converting any nested object into an array, thereby creating the required data schema.

The example below shows a deeply nested form with nested objects each with its own attributes:

![Nested form](/assets/img/nested-form/form4.png)

{% highlight json linenos %}
{
    "params":[{
        "name":"OBJ1",
        "type":"6",
        "objects":[{
            "name":"OBJ2",
            "type":"6",
            "objects":[{
                "name":"inner","type":"1"
            }]
            },
            {
                "name":"OBJ3",
                "type":"6",
                "objects":[{
                    "name":"INNER","type":"1"
                },
                {
                    "name":"INNER2",
                    "type":"1"
                }]
            }
        ]
    }]
}
{% endhighlight %}

Given the above approach, I was able to create a form with arbitrary nested objects. The UI could do with some improvements on the alignment of the nested form items.

![Nested form](/assets/img/nested-form/form5.png)

In summary, it's possible to create a nested form with pure javascript without the need for using a framework. It requires some thought on the UI design and how to create the data structure efficiently.

In further posts, I will demonstrate how the form component fits into the application which uses Gemini API to create [Function Calling in Gemini] or [Structured Output in Gemini].

H4PPY H4CK1NG !