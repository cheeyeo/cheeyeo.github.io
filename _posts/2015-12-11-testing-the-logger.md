---
layout:     post
show_meta: true
title:      Testing Logger in Ruby
header:     Testing Logger in Ruby
date:       2015-12-11 00:00:00
summary:  A more flexible approach of testing exceptions being logged in Ruby
categories: ruby testing
author: Chee Yeo
---

I came across an interesting dilemma while working on a recent project. I had a logger object within a class which writes error messages to an external log file.

While I normally don't test log files, there was a long formatted message produced by the logger which warrants some test coverage since it is triggered when certain error conditions occur, which means it is also dependent on the kind of StandardErrors that occur.

My first approach was to stub out the 'error' method on the logger object to return a mock response using mocha with MiniTest and asserting against the message when certain errors are raised.

However this apporach does not feel I'm testing the SUT but rather defining a canned response which will have to change if the underlying system implementation changes in the future.

I decided to abandon the mock/stub approach. I created a new class object called TestLogger which responds to the 'error' method call and is simply as follows:


{% highlight ruby linenos %}
class FakeLogger
  attr_accessor :errors

  def initialize
    @errors = []
  end

  def error(exception)
    @errors << exception
  end
end
{% endhighlight %}

Within the class which has the logging, I also refactor for it to accept an optional Logger as a parameter:

{% highlight ruby linenos %}
class ExceptionalClass
  # rest of class implementation

  def initialize(logger: Logger.new("mylogs.log"))
    @logger = logger
  end

  def raise_an_error!
    raise StandardError, "An error has occured"
  rescue => e
    logger.error "An exception was raised: #{e.message} #{e.class}"
  end
end

{% endhighlight %}

Within my unit tests, I just instantiate the FakeLogger class and pass it as a params into the logger method:

{% highlight ruby linenos %}
  test "it should log error messsages" do
    fake_logger = FakeLogger.new
    exceptional_class = ExceptionalClass.new(logger: fake_logger)

    assert_raise StandardError do
      exceptional_class.raise_an_error!
    end

    assert_includes fake_logger.errors, "An exception was raised: An error has occured StandardError"
  end

{% endhighlight %}

I was able to create assertions and test it against the fake logger through its @errors array in the test environment. In the production environment I am also able to pass in a custom logger of my choice or leave it as the default. I feel that this approach is more flexible to change and also provide adequate test coverage on rescuing the exceptions being raised.

Happy Hacking!
