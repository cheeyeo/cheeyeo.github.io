---
layout:     post
show_meta: true
title:      Errno::EMFILE with Paperclip
header:     Errno::EMFILE with Paperclip
date:       2016-06-01 10:00:00
summary:  How to resolve Errno::EMFILE when using Paperclip in Rails
categories: ruby rails paperclip
author: Chee Yeo
---

In a commercial project, I recently came across an unusual bug which I have never seen before. What better way to document this for furture reference and what not to do in case it ever crops up again.

The issue lies in a Rails application which uses Paperclip to store attachments in the database. The model in question uses Paperclip as per documented without any special configurations.

One fine day, the application suddenly stopped working after a bulk upload of attachments. The error logs keep reporting the following error:

{% highlight ruby %}
Errno::EMFILE: Too many open files
{% endhighlight %}

After digging through the source code, I still could not work out where the issue lie. The stacktrace did not point to where exactly the error originated from. The only clue I had was `Too many open files`. With that, I started to inspect the model which handled file uploads more closely.

There are several helper methods within this model which deserializes the binary blob from the database and calls `Paperclip.io_adapters.for` to extract metadata about the attachment such as width, height, content type.

`Paperclip.io_adapters.for` in turn calls `Paperclip::AttachmentAdapter.new` which in turns calls its `cache_current_values` method, which in turn invokes `copy_to_tempfile` with the target file. The method is shown below:

{% highlight ruby linenos %}
def copy_to_tempfile(source)
  if source.staged?
    FileUtils.cp(source.staged_path(@style), destination.path)
  else
    source.copy_to_local_file(@style, destination.path)
  end
  destination
end
{% endhighlight %}

The method is actually deserializing the binary blob from the database and saving it as a tempfile in /tmp directory. The issue here is that there is no automatic cleanup of the temp files once processing is completed. In the above case, the model keeps creating a temp file object everytime is calls `Paperclip.io_adapters.for`. This results in the temp directory being filled up as in the case of a bulk upload.

To resolve this issue we need to be able to unlink or delete the tempfile after each call to the adapter. The problem here is that the `@tempfile` instance variable is not in the adapter's public api.

I tried to use refinements on the PaperClip::AbstractAdapter class in order to make `@tempfile` readable but it did not work in the context of a Rails app due to scope issues. With a little bit of meta programming within an initializer, I came up with the following:

{% highlight ruby linenos %}
PaperClip::AbstractAdapter.class_eval do
  attr_readable :tempfile
end
{% endhighlight %}

Now, I am able to access the tempfile and close it once processing is done like so:

{% highlight ruby linenos %}
adapter = Paperclip.io_adapter.for(file)
# get the image width, height etc
width = Paperclip::Geometry.from_file(adapter).width.to_i
height = Paperclip::Geometry.from_file(adapter).height.to_i

# close the adpater and removes the temp file
adapter.tempfile.close(true) if adapter.tempfile
{% endhighlight %}

We first close and then unlink / delete the tempfile. `Tempfile.close` does it automatically when you pass true to it.

This has been a really interesting bug to track down and resolve and I did learn a lot about how Paperclip works under the hood. It also throws open my assumptions that the gem would undertake all the cleanup for me automatically. If anything, I learnt not to take for granted file IO in Ruby and always make sure that any file handles are opened and closed properly everytime.

Happy Hacking!!
