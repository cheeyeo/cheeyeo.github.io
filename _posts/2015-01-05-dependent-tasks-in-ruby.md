---
layout:     post
show_meta: true
title:      Running dependent tasks in Ruby with TSort
header:     Running dependent tasks in Ruby with TSort
date:       2015-01-05 18:04:00
summary:  How to run dependent tasks in Ruby with TSort
categories: ruby tsort
author: Chee Yeo
---

I was intrigued to learn more about the [TSort](http://www.ruby-doc.org/stdlib-2.1.2/libdoc/tsort/rdoc/TSort.html){:target="_blank"} module after reading about it from a blog post.

In summary, TSort implements toplogical sorting and determine the order to run a list of dependencies. You may have come across such scenarios when using tools such as Bundler
and Rake.

After using other task runners such as Grunt.js I wonder if I could implement
a very simple task runner in Ruby using TSort:

{% highlight ruby linenos %}
require 'tsort'

TASKS={
  :one => [],
  :two => [],
  :three => [],
  :four => [:one, :three],
  :five => [:four, :two]
}

def one
  puts "Task one ran!"
  command "date '+TASK 1 DATE: %m/%d/%y%nTASK 1 TIME:%H:%M:%S' > t1"
end

def two
  puts "Task two ran!"
  command "date '+TASK 2 DATE: %m/%d/%y%nTASK 2 TIME:%H:%M:%S' > t2"
end

def three
  puts "Task three ran!"
  command "date '+TASK 3 DATE: %m/%d/%y%nTASK 3 TIME:%H:%M:%S' > t3"
end

def four
  puts "Task four ran!"
  command 'cat t1 t3 > t4'
end

def five
  puts "Task five ran!"
  command 'cat t4 t2 > t5'
end

def command(arg)
  puts "Running command: #{arg}"
  system arg
  puts
end

def task_to_run
  yield :five
end

def dependent_tasks(t, &block)
  TASKS[t].each(&block)
end

each_node = method(:task_to_run)
each_child = method(:dependent_tasks)

puts "Running tasks in the following order:"

puts TSort.tsort(each_node, each_child)

puts

TSort.each_strongly_connected_component_from(:five, each_child) {|scc|
  method(scc.first).call
}
{% endhighlight %}

In the example, the TASKS hash constant defines the list of tasks that needs to be
completed. The key refers to the task name and its value is a list of dependent
tasks that needs to be completed before it can run.

For example, TASKS[:one] is an empty array which means it has no dependencies
and can be run first. TASKS[:four] is dependent on the completion of tasks :one
and :three. Each hash key is a symbol which corresponds to a local method.

We require 'tsort'. Then we invoke the module method [each_strongly_connected_component_from](http://www.ruby-doc.org/stdlib-2.1.2/libdoc/tsort/rdoc/TSort.html#method-c-each_strongly_connected_component_from){:target="_blank"}, which takes two arguments:
the starting node, which in this case, is the task :five from the TASKS hash.

The second argument must be an object which responds to call. 'each_child' is a method object
which references the 'dependent_tasks' method. The dependent_tasks method yields
the children of each node into a block which is processed by TSort.

[tsort](http://www.ruby-doc.org/stdlib-2.1.2/libdoc/tsort/rdoc/TSort.html#method-i-tsort){:target="_blank"} returns a list of sorted nodes from children to parents.

If we run the above script, this is the output we see, with task :five set as the default:

{% highlight ruby linenos %}
Running tasks in the following order:
one
three
four
two
five

Task one ran!
date '+TASK 1 DATE: %m/%d/%y%nTASK 1 TIME:%H:%M:%S' > t1

Task three ran!
date '+TASK 3 DATE: %m/%d/%y%nTASK 3 TIME:%H:%M:%S' > t3

Task four ran!
cat t1 t3 > t4

Task two ran!
date '+TASK 2 DATE: %m/%d/%y%nTASK 2 TIME:%H:%M:%S' > t2

Task five ran!
cat t4 t2 > t5
{% endhighlight %}

From the output, TSort sees that TASK[:five] is dependent on tasks :four and :two.
Task :four has two children: tasks :one and three. Hence, tasks :one and :three
are executed first, followed by task :four. It proceeds to run task :two and since
it has no child nodes, task :five is then run.

If any of the tasks encounter an exception, execution stops at that point
and no further tasks are run. For example, if an error is raised in task one method
call:

{% highlight ruby linenos %}
def one
  puts "Task one ran!"
  raise "Error!"
  command "date '+TASK 1 DATE: %m/%d/%y%nTASK 1 TIME:%H:%M:%S' > t1"
end
{% endhighlight %}

none of the other tasks are executed:

{% highlight ruby linenos %}
Task one ran!
tasks.rb:13:in `one': Error! (RuntimeError)
  from tasks.rb:62:in `call'
  from tasks.rb:62:in `block in <main>'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:420:in `block (2 levels) in each_strongly_connected_component_from'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:420:in `block (2 levels) in each_strongly_connected_component_from'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:429:in `each_strongly_connected_component_from'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:419:in `block in each_strongly_connected_component_from'
  from tasks.rb:53:in `each'
  from tasks.rb:53:in `dependent_tasks'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:413:in `call'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:413:in `each_strongly_connected_component_from'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:419:in `block in each_strongly_connected_component_from'
  from tasks.rb:53:in `each'
  from tasks.rb:53:in `dependent_tasks'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:413:in `call'
  from /Users/chee/.rvm/rubies/ruby-2.2.0/lib/ruby/2.2.0/tsort.rb:413:in `each_strongly_connected_component_from'
  from tasks.rb:61:in `<main>'
{% endhighlight %}

Although one could have structured the above using just arrays, TSort automatically
works out where it should start from, rather than having to work it out manually.
