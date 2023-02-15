---
layout:     post
show_meta: true
title:      Using Jekyllrb with Grunt
header:     Using Jekyllrb with Grunt
date:       2015-01-02 09:48:30
summary:  Building cheeyeo.uk using Jekyllrb with Grunt
categories: jekyllrb grunt
author: Chee Yeo
---

While building my personal blog [http://www.cheeyeo.uk](http://www.cheeyeo.uk),
I made the decision to explore other blogging frameworks outwith of the traditional
MVC framework structure.

After reading an article about [Jekyllrb](http://jekyllrb.com){:target="_blank"}
on the [Thoughtworks Technology Radar](http://www.thoughtworks.com/radar/techniques/static-site-generators){:target="_blank"}, I made the decision
to learn more about Jekyllrb by building this site using it.

One of the features lacking in using a static site generator is asset management.
In Rails we have the Asset pipeline which carries out the compilation and minifcation
process. Whilst using Jekyll I decided to roll my own using Grunt.js.

### Step 1: Structuring the assets

I would need to structure my assets in such a way that ONLY the required
files are included into the final build. In Jekyllrb, any directory can be included
into the final ___site__ folder if it is not excluded from ___config.yml__.

Rather than having to manually update the config file, I learnt that any folder
which starts with an underscore will be automatically excluded from the ___site__
folder. I created a ___assets__ folder which will hold all my sass and js files.
A ___tmp__ directory is also created as a temporary holding area which will be made
clear below.


### Step 2: Making the gruntfile

The main grunt tasks I would require to handle the assets for this site includes the following:

1. Compiling the javascript and sass files
2. Minifying / compressing the assets for production

There are already many grunt plugins available, such as grunt-contrib-concat, grunt-contrib-uglify and grunt-contrib-sass.

Below is a snippet of my Gruntfile.js for handling js and sass files:

{% highlight javascript linenos %}
  grunt.initConfig({
    pkg: grunt.file.readJSON("package.json"),

    concat: {
      options: {
        separator: ';',
      },
      dist: {
        src: ['_assets/js/*.js','!_assets/js/modernizr.*'],
        dest: '_tmp/site.js'
      }
    },

    uglify: {
      my_target: {
        files: {
          '_tmp/site.min.js': ['_tmp/site.js']
        }
      }
    },

    sass: {
      global: {
        options: {
          style: "compressed"
        },
        files: {
          "_tmp/global.css": "_assets/scss/global.scss"
        }
      }
    },
    ...
   });
{% endhighlight %}

The concat task essentially joins the required js files together from _assets/js
into _tmp/site.js which the uglify task picks up to generate the compressed js
file into _tmp/site.min.js. Likewise, the sass task joins and compresses the css
into _tmp/global.css

### Step 3: Adding the assets to the build process

The trickiest task is to combine the Grunt.js workflow into the Jekyllrb structure.

I came across the [jekyll-minibundle](https://github.com/tkareine/jekyll-minibundle){:target="_blank"} which both minifies and generates asset fingerprinting. Since the
above Grunt tasks already does minification I am only using the asset fingerprinting.

This would allow me to cache my assets using __Rack::StaticCache__ in my config.ru file. ( More on deploying your jekyllrb site in a separate article !)

Within the partials, I made the following declarations:

{% highlight html %}
{% raw  %}
<link rel="stylesheet" href="{% ministamp _tmp/global.css /assets/global.css %}" type="text/css">
{% endraw %}
{% endhighlight %}


__ministamp__ is a command from [jekyll-minibundle](https://github.com/tkareine/jekyll-minibundle){:target="_blank"}. It generates a fingerprint of ___tmp/global.css__ and moves into ___site/assets/global.css__ once the site is being built.

If ___tmp/global.css__ were to change, the fingerprint will change which means
you get automatic asset cache expiry, just like the asset pipeline!

### Conclusion

Jekyllrb is a very versatile system and there are many variations one can use to handle assets. Coupled with Grunt.js which also has a healthy ecosystem of plugins
and tasks ready to use, handling assets in static websites need not be difficult.





