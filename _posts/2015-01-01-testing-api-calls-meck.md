---
layout:     post
show_meta: true
title:      Testing API Web Calls in Elixir using meck
header:     Testing API Web Calls in Elixir using meck
date:       2015-01-01 09:48:30
summary:    How to test API calls in Elixir using Meck
categories: elixir unit test meck api
author: Chee Yeo
---

Elixir is a powerful functional programming language with an equally powerful test suite call ExUnit. it is possible to write unit and high level integration specs while developing.

As a Rubyist, one of the things that intrigued me was how do you test API calls as one could do in Ruby. I use WebMock a lot in Ruby tests - so is there something similar in Elixir / Erlang?

In a recent elixir project, I needed to do something similar and while reading through the test code for HTTPoison, I noticed a library called [meck](https://github.com/eproxus/meck){:target="_blank"} being used. Its a mocking library developed in Erlang. The following is a gist of an example of mocking api calls in Elixir:

{% highlight elixir %}
# https://github.com/eproxus/meck
# add meck as a dependency in mix.exs

defmodule GithubIssuesTest do
  use ExUnit.Case
  import :meck

  setup_all do
    new(Issues.GithubIssues)
    on_exit fn -> unload end
    :ok
  end

  test "successful fetch with user and a project" do
    # already decoded from raw json using jsx in pipeline
    mock_response = {:ok, [
      [
        {"url", "https://api.github.com/repos/elixir-lang/elixir/issues/2956"},
        {"html_url", "https://github.com/elixir-lang/elixir/issues/2956"},
        {"id", 52467010},
        {"number", 2956}
        ]
      ]
    }

    # use meck to create a mock response
    expect(Issues.GithubIssues, :fetch, fn("elixir-lang", "elixir") -> mock_response end)

    # calls the actual function which makes the actual api call!
    # expect(Issues.GithubIssues, :fetch, fn("elixir-lang", "elixir") -> :meck.passthrough(["elixir-lang", "elixir"]) end)


    assert validate(Issues.GithubIssues)

    assert Issues.GithubIssues.fetch("elixir-lang", "elixir") == mock_response
  end
end
{% endhighlight %}

The module under test makes an API call to the Github api to fetch the latest issues for a given project.

After adding meck as a dependency, I import it into the test using import :meck.

In the setup block, I created a new mock object using

{% highlight elixir %}
:meck.new(Issues.GithubIssues)
{% endhighlight %}

The on_exit call unloads :meck and restores the module its original functionality.

Within the test block, I stubbed out the actual method call using:

{% highlight elixir %}
:meck.expect(Issues.GithubIssues, :fetch, fn("elixir-lang", "elixir") -> mock_response end)
{% endhighlight %}

Every call to Issues.Github.fetch("elixir-lang","elixir") will return the mock response and will not make the actual api call.

Like in WebMock, we can force the module to make the actual API call using:

{% highlight elixir %}
:meck.passthrough(["elixir-lang", "elixir"])
{% endhighlight %}

Happy Hacking!
