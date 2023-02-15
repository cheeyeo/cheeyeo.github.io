---
layout:     post
show_meta: true
title:      Local tasks in Ansible playbook
header:     Local tasks in Ansible playbook
date:       2016-05-14 10:00:00
summary: Running local tasks in Ansible playbook
categories: ansible automation deployment
author: Chee Yeo
---

Sometimes while running ansible playbooks, I require a certain role to run some task locally before I can hand its output to be passed to a remote task. This may include processing some files in a certain manner before deployment. I tend to do these tasks manually if I remember to do it! I have lost hours debugging a failed playbook error only to realise afterwards I forgot to run these local preprocessing tasks first.

An ideal scenario would be to have roles that can execute commands locally.

In Ansible it is possible to specify a task to be run locally only. There are two ways of doing this: using the __delegate_to__ or the __local_action__ keyword.

The __delegate_to__ keyword changes or specifies the host on which to run the task. If we set this to __localhost__, it will only run locally. No other config changes are required.

For example, to rebuild all the posts in my jekyll blog:

{% highlight yaml linenos %}
---
- name: Publish Posts
  run_once: true
  delegate_to: 127.0.0.1
  command: chdir={{ jekyll_dir }} bundle exec jekyll build --config _config.yml
{% endhighlight %}

We can condense the above into a one-liner using __local_action__:

{% highlight yaml linenos %}
---
- name: Publish Posts
  run_once: true
  local_action: command chdir={{ jekyll_dir }} bundle exec jekyll build --config _config.yml
{% endhighlight %}

The __run_once__ ensures that the role is run only once.

More information on [Delegation]{:target=>:blank} can be found on the Ansible Documentation website.

Happy Hacking!

[Delegation]: http://docs.ansible.com/ansible/playbooks_delegation.html
