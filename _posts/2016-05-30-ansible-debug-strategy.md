---
layout:     post
show_meta: true
title:      Debugging Ansible playbooks
header:     Debugging Ansible playbooks
date:       2016-05-30 10:00:00
summary:  Using the debug strategy in ansible 2.1 to debug playbooks
categories: ansible deployment automation
author: Chee Yeo
---

In the latest release of Ansible 2.1, a new `debug` strategy is added which allows debugging from within each playbook when run. This launches an actual debug console which allows one to inspect variables and the running task. Think of `byebug` or `debugger` in ruby but for ansible.

As an example, consider the sample playbook below:

{% highlight yaml %}
---
- name: "Test debug"
  hosts: local
  strategy: debug
  gather_facts: no
  vars:
    var1: value1
  tasks:
    - name: Non existent variable
      ping: data={% raw %}{{ not_exists }}{% endraw %}
{% endhighlight %}


We have a simple playbook which calls a non-existent variable 'not_exists' which gets passed to the ping command. Note that we set `strategy` to be in debug mode. There are other playbook strategies which can be found here: [Ansible Strategy]{:target=>:blank}.

Once we run the playbook, when it throws an exception it drops into a debugger console like the following:

{% highlight bash %}
PLAY [Test debug] **************************************************************

TASK [Non existent variable error] *********************************************
fatal: [localhost]: FAILED! => {"failed": true, "msg": "'non_existent' is undefined"}
Debugger invoked
(debug)
{% endhighlight %}

We can also print the current result of the task by calling `p result`:

{% highlight bash %}
{'failed': True, 'msg': u"'wrong_var' is undefined"}
{% endhighlight %}

The error indicates that the current task has failed because the variable 'non_existent' is undefined. It should have been defined as 'var1'.

To change this we can fix the error directly in the playbook itself and re-run it. We can also make the changes directly in the debug console by changing the task variable to the correct one:

{% highlight bash %}
(debug) p task.args
{u'data': u'{{ non_existent }}'}
(debug) task.args['data']='{{var1}}'
(debug) p task.args
{u'data': '{{var1}}'}
(debug) redo
{% endhighlight %}

Calling `redo` continues to execute the task till the end and now we have a passing task.

Other commands we can issue in the debugger include `q` to quit and `p` print statements for vars, host, task and result.

You can read more about it on the [Ansible Debugger]{:target => '_blank'} page.

I find this feature tremendously helpful when developing playbooks locally.

Happy Hacking!

[Ansible Strategy]: http://docs.ansible.com/ansible/playbooks_strategies.html
[Ansible Debugger]: http://docs.ansible.com/ansible/playbooks_debugger.html
