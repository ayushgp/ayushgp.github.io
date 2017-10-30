---
layout: post
title: Building an image crawler using Python and Scrapy
comments: true
---
Have you ever needed to pull data from a website that doesn't provide an API? Well, you could just pull out the data from the HTML then! This tutorial will teach you how to scrape websites so that you can get the data you want from third party websites without using APIs.

Scrapy is an open source web scraping and crawling framework written in python. We'll learn how to use scrapy to crawl and scrape websites. 

## Prerequisites
You should be comfortable writing code in Python. You should also know how to use Regular Expressions(Regex). A great tutorial for learning Regex can be found on [Regexone](https://regexone.com/).

## Installation

You need the following tools:
- [Python 2.7](https://www.python.org/download/releases/2.7/)
- [Scrapy](https://doc.scrapy.org/en/latest/intro/install.html)

### Windows users
Once you have installed both python and scrapy, make sure you have them in your `PATH` environment variable. Here is a [detailed installation guide](https://scraper24x7.wordpress.com/2016/03/19/how-to-install-scrapy-in-windows/) for both python and scrapy.

## Creating a project
Once you've set up the above tools, you are ready to dive into creating a Crawler. Lets start by creating a Scrapy project. Fire up your terminal and enter: 
```
$ scrapy startproject imagecrawler
```

This will create a directory for you with the following structure. 
```
imagecrawler/
    scrapy.cfg            # deploy configuration file
    imagecrawler/             # project's Python module, you'll import your code from here
        __init__.py
        items.py          # project items definition file
        pipelines.py      # project pipelines file
        settings.py       # project settings file
        spiders/          # a directory where you'll later put your spiders
            __init__.py
```
## Building your first spider
Spiders are classes that you define and that Scrapy uses to scrape information from a website (or a group of websites). They must subclass `scrapy.Spider` and define the initial requests to make, optionally how to follow links in the pages, and how to parse the response to extract data.

Create a new file called pexels_scraper.py in the spiders folder with the following content:
```python
import scrapy

class PexelsScraper(scrapy.Spider):
    name = "pexels"
    
    def start_requests(self):
        url = "https://www.pexels.com/"
        yield scrapy.Request(url, self.parse)

    def parse(self, response):
        print response.url, response.body
```
Lets look at what the literals in the above code mean:
- `name`: identifies the Spider. You'll use this name to start crawling.
- `start_requests()`: returns an iterable of Requests that'll get executed.
- `parse()`: parses the response, extracting the scraped data as dicts and also finding new URLs to follow and creating new requests (Request) from them.

To run the code we wrote above, open your terminal and `cd` to the imagecrawler directory and enter the following command:
```
$ scrapy crawl pexels
```
This will start the crawler and print the url and the body of the response it got back. Then the crawler will stop. This is because we haven't yet specified how to move to links it encounters on a page. We'll look at that in the next section.

## Recursively crawling the website
Now that we've set up the project, let's look at the website we'll scrape. We'll scrape [Pexels](https://www.pexels.com/), a website that provides high quality and completely free stock photos. They have an API but it has a limit of 200 requests per hour. We'll crawl this website for images, url of the page we found them on and the tags associated with them.

Lets go to [Pexels.com](https://www.pexels.com/) and open an image. Let us first examine the URL structure which is used by pexels for each image. It is of the form: 
```
https://www.pexels.com/photo/cosmos-dark-galaxy-hd-wallpaper-173383/
```
All links containing photos have the following in common:
- They start with `https://www.pexels.com/photos/`
- They have an id at the end of the link: `173383`
 
We'll extract all links from the pages we visit. Then we'll filter out all links that do not match the given prefix. We'll use the id to keep track of the links we've already visited so that we do not crawl same pages repeatedly. 

First import 3 modules we'll need for the above tasks: `re`, `LinkExtractor` and `Selector`. Then we need a regex url matcher that'll match the common url. We also need a function to extract image ids from urls. Modify the PexelScraper class so that it looks like the following: 
```python
import scrapy
import re
from scrapy.linkextractor import LinkExtractor
from scrapy.selector import Selector

class PexelsScraper(scrapy.Spider):
    name = "pexels"
    
    # Define the regex we'll need to filter the returned links
    url_matcher = re.compile('^https:\/\/www\.pexels\.com\/photo\/')
    
    # Create a set that'll keep track of ids we've crawled
    crawled_ids = set()
    
    def start_requests(self):
        url = "https://www.pexels.com/"
        yield scrapy.Request(url, self.parse)

    def parse(self, response):
        body = Selector(text=response.body)
        link_extractor = LinkExtractor(allow=PexelsScraper.url_matcher)
        next_links = [link.url for link in link_extractor.extract_links(response) if not self.is_extracted(link.url)]
        
        # Crawl the filtered links
        for link in next_links:
            yield scrapy.Request(link, self.parse)
            
    def is_extracted(self, url):
        # Image urls are of type: https://www.pexels.com/photo/asphalt-blur-clouds-dawn-392010/
        id = int(url.split('/')[-2].split('-')[-1])
        if id not in PexelsScraper.crawled_ids:
            PexelsScraper.crawled_ids.add(id)
            return False
        return True
```

## Looking at the website structure
We are now getting only the pages we wanted to get. We now want to get the image urls and associated tags for the images. For this, We need to take a look at how the HTML pages for images look. Go to an [image page on pexels](https://www.pexels.com/photo/cosmos-dark-galaxy-hd-wallpaper-173383/). Now right click on the image and click Inspect Element, you'll see something like this: 

![img_class](https://cloud.githubusercontent.com/assets/7992943/25837283/b5230794-34aa-11e7-84c1-4c14a2b50634.jpg)

We can see that the `img` tag has a class `image-section__image`. We'll use this information to extract this tag. The url of the image is in the `src` attribute and the tags we need are there in the `alt` attribute. Now let us modify our `PexelsScraper` class to extract these things and print them out to the console. 

For this we'll create 2 regex patterns that'll extract the src and alt attributes for us. Then we'll use the `css` method in the `Selector` class to extract `img` tags with class `image-section__image`. Finally we'll extract the url and tags and print them to the screen.

Add the following variables to the PexelsScraper class:
```python
src_extractor = re.compile('src="([^"]*)"')
tags_extractor = re.compile('alt="([^"]*)"')
```

Now modify the `parse` method such that it prints the required url and tags: 
```python
def parse(self, response):
    body = Selector(text=response.body)
    images = body.css('img.image-section__image').extract()
    
    # body.css().extract() returns a list which might be empty
    for image in images:
        img_url = PexelsScraper.src_extractor.findall(image)[0]
        tags = [tag.replace(',', '').lower() for tag in PexelsScraper.tags_extractor.findall(image)[0].split(' ')]
        print img_url, tags

    link_extractor = LinkExtractor(allow=PexelsScraper.url_matcher)
    next_links = [link.url for link in link_extractor.extract_links(response) if not self.is_extracted(link.url)]
    
    # Crawl the filtered links
    for link in next_links:
        yield scrapy.Request(link, self.parse)
```

The body.css('img.image-section__image').extract() call gives us all the `img` tags with class `image-section__image` in a list.

Now you can run the spider and test it out! Open your terminal and enter the following:
```
$ scrapy crawl pexels
```

You'll get the output similar to the following: 
```
https://images.pexels.com/photos/132894/pexels-photo-132894.jpeg?w=940&amp;h=650&amp;auto=compress&amp;cs=tinysrgb [u'red', u'and', u'grey', u'fish', u'on', u'ice']
https://images.pexels.com/photos/343812/pexels-photo-343812.jpeg?w=940&amp;h=650&amp;auto=compress&amp;cs=tinysrgb [u'cuisine', u'delicious', u'diet']
https://images.pexels.com/photos/274595/pexels-photo-274595.jpeg?w=940&amp;h=650&amp;auto=compress&amp;cs=tinysrgb [u'adult', u'beautiful', u'beauty']
https://images.pexels.com/photos/230824/pexels-photo-230824.jpeg?w=940&amp;h=650&amp;auto=compress&amp;cs=tinysrgb [u'orange', u'squash', u'beside', u'stainless', u'steel', u'bowl']
```

## Wrapping up
So in around 50 lines of code, we were able to get a web crawler( which scrapes a website for images) up and running. This was just a tiny example of something you could do with a web crawler. There are whole businesses running based on web scraping, for example, most of the product price comparison websites use crawlers to get their data. 

Now that you have the basic knowledge of how to build a crawler, go and try building your own crawler!
