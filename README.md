### cheeyeo.dev website

Public website for https://www.cheeyeo.dev

#### ON running rvm on Ubuntu 22.04

Major issue with openssl causing rvm install not to work

Workaround:

```
$rvm pkg install openssl
$rvm install ruby-<version> --with-openssl-dir=/usr/share/rvm/usr
```

To fix __rvm make install error:
```
rvmsudo rvm get head
```

Ref: https://github.com/rvm/rvm/issues/5216#issuecomment-1206488598



#### To run locally

* Setup ruby:
```
rvm use ruby-3.0.0 && \
rvm gemset use jekyll2
```

* Running on localhost:
```
bundle exec jekyll serve --baseurl=
```

* Generate using jekyll:
```
bundle exec jekyll build --config _config.yml
```

### NOTE ON USING DOUBLE CURLY BRACES

If you have double curly braces in a code block using `highlight` will throw a Liquid render error. Ensure that you wrap it in `raw` block within the `highlight` block:

```
{% highlight html%}
{% raw %}
     <h2> {{ user.name.first | uppercase }}</h2>
     <p> {{ user.email }}</p>
{% endraw %}
{% endhighlight %}
```

https://stackoverflow.com/questions/24102498/escaping-double-curly-braces-inside-a-markdown-code-block-in-jekyll


### Ref

[How to use / generate Jekyll plugins]: https://learn.cloudcannon.com/jekyll/using-jekyll-plugins/

* [How to use / generate Jekyll plugins]