---
layout: post
show_meta: true
title: Trying out the Template Design pattern
header: Trying out the Template Design pattern
date: 2024-09-06 00:00:00
summary: Overview of the Template Design Pattern
categories: python design-pattern template
author: Chee Yeo
---

In a recent project for web scraping, I created a group of web scrapers that would parse a target site content based on specific html tags. However, the presence of these tags vary across various sites. I developed a basic algorithm to process the sites:

* Make a GET request to the target using `requests` library
* Pass the returned HTML response to `BeautifulSoup` library for parsing
* Look for script tag of `application/ld+json` as it may contain schema information
* If none found, look for `og:meta` tags as it may contain site information

I wrapped the above algorithm in a top level runner class and created separate scraper classes which I pass as arguments into the runner during runtime.

However, I noticed I had to duplicate some of the common functions across the various scraper classes such as creating separate HTTP requests objects and loading the HTML body into BeautifulSoup.

I started to investigate the Template design pattern as an option for refactoring. The Template design pattern is a behavioural pattern which allows you to define the algorithm in a base class and let the subclasses override the algorithm if required. 

Given the use case above, we could wrap the entire algorithm into a base class which implements the core algorithm steps:

{% highlight python %}
from abc import ABC, abstractmethod
import re
from typing import Dict, Any
import json
import requests
from bs4 import BeautifulSoup


class Scraper(ABC):
    def parse_site(self, url: str, headers: dict={}) -> Dict[str, Any]:
        self.res = {}
        self.get_html(url, headers)
        self.parse_ld_tags()
        if len(self.res) < 1:
           self.parse_meta_tags()
        
        # can be overridden in subclasses
        self.parse_custom()

        return self.res
    
    def get_html(self, url: str, headers: dict={}) -> None:
        try:
            req = requests.get(url, headers=headers)
            req.raise_for_status()
            self.html_body = req.text
            self.soup_obj = BeautifulSoup(req.text, "lxml")
        except requests.exceptions.RequestException as e:
           print(str(e))
        
    def parse_ld_tags(self) -> None:
       tags = self.soup_obj.find_all("script", type="application/ld+json")
       
       if len(tags) > 0:
        print('Processing the ld tags and filling res dict...')
        site_json = [x for x in tags if json.loads(x.text)["@graph"][0]["@type"] == "WebPage"][0]
        site_json = json.loads(site_json.text)
        self.res["title"] = f"TITLE FROM LD TAGS - {site_json["@graph"][0]['name']}"
        self.res["description"] = f"DESCRIPTION FROM LD TAGS - {site_json["@graph"][0]['description']}"

    def parse_meta_tags(self) -> None:
        metas = self.soup_obj.find_all("meta", property=re.compile("og:*"), content=True)
        if len(metas) > 0:
            print('Processing meta og tags...')
            for meta in metas:
               match meta.get("property"):
                case "og:title":
                    name = meta.get("content")
                    self.res["title"] = f"TITLE FROM META TAGS - {name}"
                case "og:description":
                     description = meta.get("content")
                     self.res["description"] = f"DESCRIPTION FROM META TAGS - {description}"


    # Optional method to override to add custom scraping rules
    def parse_custom(self) -> None:
       pass
    
{% endhighlight %}

The Scraper class implements the `parse_site` method which represents the algorithm steps. It retrieves the target site html and creates a beautifulsoup object. Then it calls `parse_ld_tags` which attempts to retrieve any application/ld+json tags if present. If not, it defaults to running `parse_meta_tags` method. Note that the `parse_custom` method can be overridden in subclasses to provide custom parsing logic.

Since `parse_custom` is blank, we can override this in custom subclasses. Assuming we need a custom scraper which only looks for title and meta-description tags by overriding the `parse_custom` method:

{% highlight python %}
class CustomScraper(Scraper):
  def parse_custom(self) -> None:
    # Look for title tags and meta description tags only
    title = self.soup_obj.find("title")
    description = self.soup_obj.find("meta", attrs={'name':'description'})
    self.res["title"] = f"TITLE FROM PARSE CUSTOM - {title.text}"
    self.res["description"] = f"DESCRIPTION FROM PARSE CUSTOM - {description['content']}"
{% endhighlight %}

Below is a full listing of the example code above:
<script src="https://gist.github.com/cheeyeo/3ed5f6191a3d7e6ff6f7ec899b1a5c44.js"></script>

While it works, there are some disadvantages in using this approach:

* The logic of the algorithm is tied to a base class. Any changes to this would require changes to the subclasses.

* Implementation of custom scraping would require creating a custom subclass and implementing a specific method.

However, the code base is a bit cleaner by pulling the duplicate code into a base class. It also allows you to overwrite part of the algorithm logic while keeping the rest of it intact.

Future posts will investigate the use of other patterns such as the Factory Method in a similar refactoring scenario.