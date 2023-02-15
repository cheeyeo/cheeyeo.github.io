---
layout:     post
show_meta: true
title:      Testing ElasticSearch with pytest
header:     Testing ElasticSearch with pytest
date:       2021-01-08 00:00:00
summary:  How to test elasticsearch document models in python with mocking
categories: python elasticsearch pytest
author: Chee Yeo
---

In a recent project, I have to improve the application's test coverage by writing test code for the data layer, which comprises of a set of document models using elasticsearch as the data storage.

From my past experiences with developing web applications, the usual approach would be to create a test copy of the database, run the tests against it, and clean up the test database once the test run is complete.

In this case, the database is running within a Docker container. While the idea of creating and running a separate container is appealing it has the following issues:

* The entire pytest suite would be dependent on having a separate process to create and run a docker container for ES. If the container setup process fails, the entire test pipeline also fails.

* The CI scripts would need to be redeveloped to cater for provisioning a separate ES container just for testing. This might not work in certain CI environments such as Github Actions.

Apart from the CI considerations, there is also another bigger concern: Are we really testing the system or are we testing the ES database? If we are only interested in the data layer interaction, we can safely mock out the parts of the system that interact with the database to return the appropriate canned responses. This would ensure that the test suite is portable and make the test intentions clearer.

There are several pypi packages for elasticsearch testing but it doesn't apply in my use case since we are running ES in a container.

Instead, I will show how I manage to mock and patch specific ES Document model methods.

Firstly, we need to create a mock client to replace the real ES client to mock out the connection. Within `conftest.py`, I created a fake ES connection client like so:

{% highlight python linenos %}
@pytest.fixture
def mock_client():
    client = Mock()
    add_connection("mock", client)
    yield client
    connections._conn = {}
    connections._kwargs = {}
{% endhighlight %}

The code above creates a mock client, and adds it to the connection pool for ES. We give it a name of "mock" since each connection in the pool needs an identifier. This will be used later with the `using` keyword.

We also need to create a separate fixture for what we expect a successful response to be. For instance, we can create a succesful GET response as follows:
{% highlight python linenos %}
@pytest.fixture
def mock_get_response():
    return {
        "found": True,
        "_type": "_doc",
        "_id": "test",
        "_index": "testindex",
        "_source": {
            "status": "RUNNING"
        },
    }
{% endhighlight %}

The fixture above returns a successful get response as an actual Elasticsearch API call would do. I think this is a clean approach as we can reuse this fixture response in different unit tests and alter the fields appropriately for the use case.

Next, we can use this set of fake client and response in our elasticsearch specific test scripts. For the sake of this article, I am assuming I have a model developed using `elasticsearch_dsl` as follows:
{% highlight python linenos %}
class Model(Document):
    status = Keyword()
    created_at = Date()

    @classmethod
    def find_or_create(cls, id, **kwargs):
        instance = None
        try:
            instance = Model.get(id=id, refresh=True, **kwargs)
        except NotFoundError as e:
            instance = TrainModel()
            instance.status = "CREATED"
            instance.meta.id = id
            instance.save(refresh=True)
        return instance
{% endhighlight %}

The `Model` definition is a standard class for an elasticsearch model. It subclasses `Document` and defines the required fields.

We also define a class method called `find_or_create` which attemmpts to first find the model; if it does not exist, it will raise the `NotFoundError` from `elasticsearch_py` package and we create and save a new model instance.

To test the above snippet, we can write a unit test like so:
{% highlight python linenos %}
@patch("elasticsearch_dsl.Document.save")
def test_find_or_create(mock_save, mock_client, mock_get_response):
    mock_save.return_value = True

    mock_client.get.return_value = mock_get_response
    
    resp = Model.find_or_create("test", using="mock")
    assert isinstance(resp, Model)
    assert resp.meta.id == "test"
    assert resp.status != "CREATED"

    # Test raise NotFound exception
    mock_client.get.reset_mock()
    mock_client.get.side_effect = NotFoundError()
    resp = Model.find_or_create("test", using="mock")
    assert resp.meta.id == "test"
    assert resp.status == "CREATED"
{% endhighlight %}

On line 1, we patch the document save method as we are not interested in making an actual database call. 

On line 2, within the test function, we import the patched save call as well as the two pytest fixtutres defined earlier.

On line 3, we create a new mock object for the `save` method as we are not interested in the success/failure of the call so we can stub its value to return True.

On line 5, we stub `mock_client.get.return_value` by setting its returned value to `mock_get_response`. This stubs the return value of `Model.get` in the class method.

On line 7, we pass the mock client to the underlying `get` call by specifying `using="mock"`. This uses the mock client fixture and returns the expected response. This is the core of the stubbing process.

How could this work? Within the `elasticsearch_dsl` package, it delegates all the `Document` calls to the underlying elasticsearch client. For instance, `Model.get` is delegated to `es.get()` where `es` is a reference to the current connection being used. Here is the [source from the elasticsearch_dsl]{:target="_blank"}

The above approach also extends to other calls. For example, if we want to test a search query call, we can rewrite it as follows:

{% highlight python linenos %}
mock_client.search.return_value = mock_response
{% endhighlight %}

With the above approach, I was able to test my elasticsearch models without requiring a dependency on an ES docker container. While the approach might seem tedious, I believe it helps make the test cases clearer since the stubs would need to define and return the appropriate mock responses.

Happy Hacking !!!

[source from the elasticsearch_dsl]: https://github.com/elastic/elasticsearch-dsl-py/blob/master/elasticsearch_dsl/document.py#L190-L206