---
layout:     post
show_meta: true
title:      Agents in Elixir
header:     Using Agents in Elixir
date:       2015-06-22 23:04:00
summary:  Using Agents in Elixir
categories: elixir
author: Chee Yeo
---

In a recent project on building a test suite similar to ExUnit in Elixir, I came across the issue of having to track or maintain state for keeping count of the number of assertions passing or failing when a tets suite is run.

I thought this would be the perfect opportunity to try out [Agent](http://elixir-lang.org/docs/v1.0/elixir/Agent.html){:target="_blank"}. From the documentation, an Agent is an abstraction around state and allows different processes to share or store the state.

Since the test assertions are defined using a macro and runs in a separate module and process, using an agent would allow the separation of concerns by allowing the reporting code to reside in a separate module and triggered when needed. The agent can then be started when the actual assertion runs and accumulates a report at the end of the testing process.

The following snippet of code is my implementation of the above:

{% highlight elixir %}
defmodule Assertion.Stats do
  @name __MODULE__

  def start_link do
    Agent.start_link(fn -> %{total: 0, passes: 0, failures: 0} end, name: @name)
  end

  def test_pass do
    Agent.update(@name, fn dict ->
      Dict.update(dict, :passes, &(&1), fn(val) -> val+1 end)
    end)

    update_test_case_count
  end

  def test_fail do
    Agent.update(@name, fn dict ->
      Dict.update(dict, :failures, &(&1), fn(val) -> val+1 end)
    end)

    update_test_case_count
  end

  def update_test_case_count do
    Agent.update(@name, fn dict ->
      Dict.update(dict, :total, &(&1), fn(val) -> val+1 end)
    end)
  end

  def count_for(term) do
    Agent.get(@name, fn dict -> Dict.get(dict,term) end)
  end

  def terms do
    Agent.get(@name, fn dict -> Dict.keys(dict) end)
  end

  # returns a new map with values reset
  def reset do
    Agent.update(@name, fn dict ->
      %{dict| passes: 0, failures: 0, total: 0}
    end)
  end

  def report do
    IO.puts """
    ===============================================
    TEST REPORT
    ===============================================
    Pass: #{Assertion.Stats.count_for(:passes)}
    Failures: #{Assertion.Stats.count_for(:failures)}
    Total test cases: #{Assertion.Stats.count_for(:total)}
    """

    Assertion.Stats.reset
  end
end
{% endhighlight %}

Within Agent.start_link, we define an initial map with the keys "passes", "failures" and "totals" to keep track of the various states of the test cases. When a test passes or fails, we call __'Assertion.Stats.test_pass'__ or __'Assertion.Stats.test_fail'__ to update the map appropriately. Then, we call __'update_test_case_count'__ to update the total count for all the test cases run. Finally, we can call __'report'__ to generate a summary of the tests and __'reset'__ simply returns a new map with the values set to 0.

Within the actual macro that defines the assertion, we can invoke the reporter like so:

{% highlight elixir %}
defmacro assert(expr, opts) do
# code to define the assertion
.....
#starts the agent
Assertion.Stats.start_link

if test_pass do
  Assertion.Stats.test_pass
else
  Assertion.Stats.test_fail
end
end

# finally after all the asertions are run
Assertion.Stats.report
{% endhighlight %}

The actual implementation can be [found here](https://github.com/cheeyeo/MetaProgramming-Elixir/blob/master/chapter2/assert_advanced1.exs)
