---
layout:     post
show_meta: true
title:      Testing Machine Learning Projects in Tensorflow 2.0+
header:     Testing Machine Learning Projects in Tensorflow 2.0+
date:       2021-03-01 00:00:00
summary:  Quick guide on how to test your machine learning projects in tensorflow 2.0
categories: tensorflow-2.0 machine-learning pytest testing
author: Chee Yeo
---

When building and developing machine learning models, one of the commonly asked questions is how do I test or verify that the model's behaviour matches its specifications.

I have seen examples of tests in various implementations on articles and open source projects. However, none of them deal with the actual question of testing model behaviour.

Its not until I encountered [Jeremy Jordan Testing ML article]{:target="_blank"} and an implementation example from [Eugene Yan Testing ML implementation]{:target="_blank"} that I grapsed the idea of the process.

In this article I will attempt to explain how I am testing my ML models in the tensorflow framework.

### Types of tests

From the articles above, ML tests can be broadly categorized into `pre-train` and `post-train` tests.

Pre-train tests are run before the actual training process starts. Such tests would include but not limited to:

* Checking that the model's output shape matches the output shape of the dataset

* Checking that a single training loop results in a decrease in loss

* Checking that the output predictions of the model falls within a specific range. For example, if the loss to minimise is `categorical_crossentropy` we expect the output to sum to 1.0

* Checking that the model can actually learn by overfitting it on the training set

* Checking for data leak in the train / test sets

If any of the above pre-train tests were to fail, it should fail and halt the training process. This is to prevent wasting any valuable resource such as GPU in the cloud.

Post-train tests are run after the model has been trained. Such tests are usually more specific but can be broadly categorized as such:

* Invariance tests: Testing that perturbation to an input should still yield the same output.

* Unit-directional tests: Testing that perturbation to an input should yield the desired output.

* Unit tests: Testing on a specific section of the dataset.

### Test Structure

Given the above tests, how do we incorporate them into an existing project? 

Most python projects that have tests normally use a test framework such as `pytest`. We can leverage it for our model tests. 

In my use case, I categorized the model tests into its respective folders in the `tests` subdirectory within a project: `pretrain`, `posttrain`

Within each of these sub directories, I further categorized these tests based on the model behaviour I'm testing for:

{% highlight console linenos %}
tests/
|- pytest.ini
|- pretrain/
  |- test_output_shape.py
  |- test_loss.py
  ...
|- posttrain/
  |- test_rotation_invariance.py
  |- test_perspective_shift.py
  ...
{% endhighlight %}

Now, given the above tests, how do we fit it within the pipeline of model training? This is where callbacks come into play...


### Use of callbacks

One of the most important features in tensorflow is the use of callbacks, which can be injected into the model's training or prediction pipeline when you call `model.fit` or `model.predict`.

We can define custom callbacks that invoke the test runs before and after training a model in order to run the pre-train and post-train tests.

These callbacks will be passed into the `model.fit` function call, which takes a `callbacks` keyword argument list.

The callback functions we want to hook into are:

* `on_train_begin`: This is called before any model training starts

* `on_train_end`: This is called after all training stops.

An example of a pre-train test custom callback can be:

{% highlight python linenos %}
from tensorflow.keras.callbacks import Callback

class PreTrainTest(Callback):
    def __init__(self, train_data, train_labels, test_data, test_labels):
        super(PreTrainTest, self).__init__()
        self.train_data = train_data
        self.train_labels = train_labels
        self.test_data = test_data
        self.test_labels = test_labels

    def on_train_begin(self):
        CustomTestRunner(
        	"tests", 
        	"pretrain", 
        	self.model, 
        	self.train_data, 
        	self.train_labels, 
        	self.test_data, 
        	self.test_labels).run()
{% endhighlight %}

Example of invoking the callback in model code:

{% highlight python linenos %}
...

model.fit(
	Xtrain, 
	ytrain, 
	validation_data=(Xtest, ytest),
	epochs=30,
	batch_size=32, 
	callbacks=[PreTrainTest(Xtrain, ytrain, Xtest, ytest)])
{% endhighlight %}

Note the use of the `CustomTestRunner` class. This will invoke `pytest` dynamically as we want to run the pre-train tests before any training begins.

An example of `CustomTestRunner`:

{% highlight python linenos %}
class CustomTestRunner:
    def __init__(self, directory, kind, model, train_data, train_labels, test_data, test_labels):
        self.directory = directory
        self.kind = kind
        self.model = model
        self.train_data = train_data
        self.train_labels = train_labels
        self.test_data = test_data
        self.test_labels = test_labels

    def run(self):
        testdir = os.path.join(self.directory, self.kind)
        sys.path.append(testdir)
        testfiles = os.listdir(testdir)

        for test in testfiles:
            """For each test file,
            we want to import it using importlib
            so we can set the model, train data etc
            as attributes
            """

            testpath = os.path.join(testdir, test)
            mod = importlib.import_module(test.split(".")[0])

            """Each test must be structured as a class
            before this will work..
            """

            for name, x in mod.__dict__.items():
                # iterate through the imported module 
                # until we find the top level class
                # left as an exercise...
                found_class = x

            setattr(found_class, "model", self.model)
            setattr(found_class, "xtrain", self.train_data)
            setattr(found_class, "ytrain", self.train_labels)
            ...


            status = pytest.main([testpath])

            if status == pytest.ExitCode:
               raise Exception("Tests run failed..")
{% endhighlight %}

The custom test runner takes in a test directory path, what kind of tests its running and the model and data attributes for the tests to run.

Each test case needs to be structured as a class as follows:

{% highlight python linenos %}
class MyPreTrainTest:
    def test_output_shape(self):
        """
        here we have access to self.model and self.xtrain
        to work on...
        """
        ...
{% endhighlight %}

The above is adopted for the post-train tests in a separate callback which uses the `on_train_end` function call instead.

This structure enables me to test my model as well as working within the tensorflow framework with the least disruption to the workflow.

Happy Hacking !!!

[Jeremy Jordan Testing ML article]: https://www.jeremyjordan.me/testing-ml/

[Eugene Yan Testing ML implementation]: https://eugeneyan.com/writing/testing-ml/