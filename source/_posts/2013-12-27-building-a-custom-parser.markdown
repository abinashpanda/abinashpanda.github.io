---
layout: post
title: "Building a custom Parser"
date: 2013-12-27 10:08
comments: true
categories: DataMining, Parsing, LXML
---
A Web crawler is an Internet bot that systematically browses the World Wide Web, typically for the purpose of Web indexing.

This tutorial is about building your own web crawler - not the one that can scan the whole internet(like [Google](www.google.com)), but one that is able to extract all the links from a given webpage. For this tutorial, I would be extracting information from IMDb.

[IMDb](www.imdb.com) is an online database of information related to films, television programs and video games. I woulb be going to parse IMDb Top 250 and IMDb: Years and extract information about the movies ratings, year of release, start casts, directors, etc.

First, let's write a simple program (hard-code) to parse [IMDb Top 250](http://www.imdb.com/chart/top).

```python main.py
#!/usr/bin/env python

from lxml.html import parse

tree = parse('http://www.imdb.com/chart/top')

movies_data = tree.findall('//*[@id="main"]/table[2]/tr/td[3]/font/a')
movies_rating = tree.findall('//*[@id="main"]/table[2]/tr/td[2]/font')

# Removing unwanted data
movies_data.pop(0)
movies_rating.pop(0)

movies_rating = [float(movie.text) for movie in movies_rating]


def get_movie_data(iterator):
    movie_data = (iterator.next(), iterator.next())
    return movie_data

mov_dict = {get_movie_data(movies_data[i].itertext())[0]:
            [int(get_movie_data(movies_data[i].itertext())[1].strip(' ()/I')),
             movies_rating[i]] for i in range(len(movies_data))}
```

Going through the code in details. For this simple parser I have used parse from lxml.html.

<code>tree = parse('http://www.imdb.com/chart/top')</code> parses the url and returns a tree.
Before going on the next line, lets discuss about XPath. XPath, the XML Path Language, is a query language for selecting nodes from an XML document. [XPath Tutorial](http://www.w3schools.com/xpath/) is a very good tutorial for XPath by w3cschools.com. In the XPath, <code>‘//*[@id=“main”]/table[2]/tr/td[3]/font/a’</code>.

<pre>
	<code>
// : Selects nodes in the document from the current node
     that match the selection no matter where they are.  
/ : Selects from the root node  
/tr/td[3]: Selects the third td element that is the child of the tr element.   
	</code>
</pre>

To get the XPath of an element, you can use Google Chrome. Click on *Inspect Element*.

{% img /images/screen1.jpg %}

Then select *Copy XPath*. This would give you the XPath to be used. **Remember to remove <tbody> element from the XPath. and also remove [] from tr as you want to scrape the whole movies list.**
Similarly you can find the XPath for movies_rating also.

Then in

``` python get_movie_data
def get_movie_data(iterator):
    """ Returns movie_name, year_of_release as movies_data[element].itertext()
    would return an iterator containing these two elements"""
    movie_data = (iterator.next(), iterator.next())
    return movie_data

mov_dict = {get_movie_data(movies_data[i].itertext())[0]:
            [int(get_movie_data(movies_data[i].itertext())[1].strip(' ()/I')),
             movies_rating[i]] for i in range(len(movies_data))}
```
*movie_dict* is built containing the dictionary with

<pre>
	<code>
key: movie_name
value: year_of_release, movie_rating
	</code>
</pre>

Let's go a step further. After writing a simple (but hard-coded) parser, I am going to write a more generic (yet simple) parser. For this, I have take some concepts from **scrapy** *(imitation is the best form of flattery)* and have used *lxml* for scrapping.

I would be scraping the [IMDb: Years page](http://www.imdb.com/year). This page contains the links for pages containing the links for Most Popular Titles Released in that year. In the next blog, I would be using the data scraped (a dictionary <code>{year: [name, rating, genres, director, actors]}</code> for analysing trends in Movies.

```python Crawler.py
import urllib2
import re
from lxml.html import parse


class Crawler():

    def __init__(self, settings):
        """
        settings should be a dictionary containing
        domain:
        start_url:

        EXAMPLE
        settings = {'domain': 'http://www.imdb.com', 'start_url': '/year'}
        """
        self.settings = settings
        self.rules = {self.settings['start_url']: 'parse'}
        self.parsed_urls = []
        self.url_list = []

    def _get_all_urls(self, response):
        """
        _get_all_urls returns all the urls in the page
        """
        tree = parse(response)
        url_list = tree.findall('//a')
        url_list = [url.attrib['href']
                    if url.attrib['href'].startswith('http://')
                    else urllib2.urlparse.urljoin(self.settings['domain'],
                                                  url.attrib['href'])
                    for url in url_list]
        return url_list

    def set_rules(self, rules):
        """
        set_rules set the rules for crawling
        rules are dictionary in the form
        {url_pattern: parsing_function}

        EXAMPLE
        >>> settings = {'domain': 'http://www.imdb.com', 'start_url': '/year'}
        >>> imdb_crawler = Crawler(settings)
        >>> imdb_crawler.set_rules({'/year/\d+': 'year_parser',
        ...                         '/title/\w+': 'movie_parser'})
        """
        self.rules = rules

    def _get_crawl_function(self, url):
        """
        _get_crawl_function returns the crawl function to be
        used for given url pattern
        """
        for pattern in self.rules.keys():
            if re.search(pattern, url):
                return self.rules[pattern]

    def parse(self, response):
        """
        parse is the default parser to be called
        """
        pass

    def start_crawl(self):
        """
        start_crawl is the method that starts calling
        
        EXAMPLE
        >>> foo_crawler = Crawler()
        >>> foo_crawler.start_crawl()
        """
        response = urllib2.urlopen(
            urllib2.urlparse.urljoin(self.settings['domain'],
                                     self.settings['start_url']))
        self.url_list = self._get_all_urls(response)
        for url in self.url_list:
            if url not in self.parsed_urls:
                crawl_function = self._get_crawl_function(url)
                if crawl_function:
                    getattr(self, crawl_function)(urllib2.urlopen(url))
                    self.parsed_urls.append(url)
```
Crawler object have to be initailized with a dictionary settings <code>{‘domain’: domain_of_\page, ‘start_url’: start_url_page}</code> The *Crawler* class has attribute url_list that contains all the urls to be parsed and parsed_urls is a list of all the parsed urls.

```python _get_all_urls
def _get_all_urls(self, response):
    """
    _get_all_urls returns all the urls in the page
    """
    tree = parse(response)
    url_list = tree.findall('//a')
    url_list = [url.attrib['href']
                if url.attrib['href'].startswith('http://')
                else urllib2.urlparse.urljoin(self.settings['domain'],
                                              url.attrib['href'])
                for url in url_list]
    return url_list
```

<code>_get_all_urls</code> retrieves all the urls present in the web page. <code>tree.findall(‘//a’)</code> would return all the //a tags present in the web page. If the url starts with http:// then it would append it as usual; but if the url is a relative url, it would append the final url formed by joining the url with the domain
<pre>
	<code>
urllib2.urlparse.urljoin(self.settings['domain'], url.attrib['href']
	</code>
</pre>

<code>parse(response)</code> is the default parser of the Crawler. More sophisticated or complex parsers can be written for different urls using <code>set_rule</code>. For example:

```python set_rule
settings = {'domain': 'http://www.imdb.com', 'start_url': '/year'}
imdb_crawler = Crawler(settings)
# year_parser is parser for scraping year pages
# movie_parser is parser for scraping movie pages
imdb_crawler.set_rules({'/year/\d+': 'year_parser',
                        '/title/\w+': 'movie_parser'})
```
All the parser should have be an input parameter <code>response</code>. Discussed below in details.

```python start_crawl
def start_crawl(self):
    """
    start_crawl is the method that starts calling
    
    EXAMPLE
    >>> foo_crawler = Crawler()
    >>> foo_crawler.start_crawl()
    """
    response = urllib2.urlopen(
        urllib2.urlparse.urljoin(self.settings['domain'],
                                 self.settings['start_url']))
    self.url_list = self._get_all_urls(response)
    for url in self.url_list:
        if url not in self.parsed_urls:
            crawl_function = self._get_crawl_function(url)
            if crawl_function:
                getattr(self, crawl_function)(urllib2.urlopen(url))
                self.parsed_urls.append(url)
```

<code>start_crawl</code> is the main function that initiates crawling. First, it tries to get all the *urls* present in the start_page. It then searches the parser to be called for that particular url using <code>_get_crawl_function</code>.

```python
def _get_crawl_function(self, url):
    """
    _get_crawl_function returns the crawl function to be
    used for given url pattern
    """
    for pattern in self.rules.keys():
        if re.search(pattern, url):
            return self.rules[pattern]
```

The particular parser is then called according to rules set above. Finally the url is appended into *parsed_urls* list.

Using the above <code>Crawler</code>, I have implemented <code>IMDbCrawler</code>.

```python main.py
from Crawler import Crawler
from collections import defaultdict

movie_final_dict = defaultdict(list)

class IMDbCrawler(Crawler):

    def year_parser(self, response):
        tree = parse(response)
        year = tree.find('//*[@id="header"]/h1')
        print year.text
        list_even = tree.findall(
            '//table//tr[@class="even detailed"]/td[@class="title"]/a')
        list_odd = tree.findall(
            '//table//tr[@class="odd detailed"]/td[@class="title"]/a')
        movies_list = list_even + list_odd
        movies_list_url = [urllib2.urlparse.urljoin(self.settings['domain'],
                                                    movie.attrib['href'])
                           for movie in movies_list]
        for url in movies_list_url:
            self.url_list.append(url)

    def movie_parser(self, response):
        tree = parse(response)
        name = tree.find('//*[@id="overview-top"]/h1/span[1]').text
        print name
        try:
            genres = tree.findall('//div[@itemprop="genre"]//a')
            genres = [genre.text.strip() for genre in genres]
            director = tree.find(
                '//div[@itemprop="director"]//span[@itemprop="name"]')
            director = director.text.strip()
            rating = tree.find('//span[@itemprop="ratingValue"]')
            rating = float(rating.text)
            actors = tree.findall('//td[@itemprop="actor"]//a//span')
            actors = [actor.text.strip() for actor in actors]
            year = tree.find('//*[@id="overview-top"]/h1/span[2]/a').text
            movie_final_dict[year].append([name, rating,
                                           genres, director, actors])
        except (AttributeError, IndexError):
            pass

settings = {'domain': 'http://www.imdb.com', 'start_url': '/year'}
imdb_crawler = IMDbCrawler(settings)
imdb_crawler.set_rules({'/year/\d+': 'year_parser',
                        '/title/\w+': 'movie_parser'})
imdb_crawler.start_crawl()
```

<code>IMDbCrawler</code> has inherited *Crawler* class.

```python year_parser
def year_parser(self, response):
    tree = parse(response)
    year = tree.find('//*[@id="header"]/h1')
    print year.text
    list_even = tree.findall(
        '//table//tr[@class="even detailed"]/td[@class="title"]/a')
    list_odd = tree.findall(
        '//table//tr[@class="odd detailed"]/td[@class="title"]/a')
    movies_list = list_even + list_odd
    movies_list_url = [urllib2.urlparse.urljoin(self.settings['domain'],
                                                movie.attrib['href'])
                       for movie in movies_list]
    for url in movies_list_url:
        self.url_list.append(url)
```
<code>year_parser</code> would crawl year pages and append the movie pages into url_list. XPath for extracting movie url and year name is discussed above.

```python movie_parser
def movie_parser(self, response):
    tree = parse(response)
    name = tree.find('//*[@id="overview-top"]/h1/span[1]').text
    print name
    try:
        genres = tree.findall('//div[@itemprop="genre"]//a')
        genres = [genre.text.strip() for genre in genres]
        director = tree.find(
            '//div[@itemprop="director"]//span[@itemprop="name"]')
        director = director.text.strip()
        rating = tree.find('//span[@itemprop="ratingValue"]')
        rating = float(rating.text)
        actors = tree.findall('//td[@itemprop="actor"]//a//span')
        actors = [actor.text.strip() for actor in actors]
        year = tree.find('//*[@id="overview-top"]/h1/span[2]/a').text
        movie_final_dict[year].append([name, rating,
                                       genres, director, actors])
    except (AttributeError, IndexError):
        pass
```
<code>movie_parser</code> would crawl the movie web page and add the details of the movies such as its *ratings, director, genres* in the dictionary *movie_final_dict*. There are some movies where data (rating, actors, etc.) is missing, so I have included try and except statements.

```python
settings = {'domain': 'http://www.imdb.com', 'start_url': '/year'}
imdb_crawler = IMDbCrawler(settings)
imdb_crawler.set_rules({'/year/\d+': 'year_parser',
                        '/title/\w+': 'movie_parser'})
imdb_crawler.start_crawl()
```
<code>imdb_crawler</code> is an instance of *IMDbCrawler*. It has been instantiated with the domain http://www.imdb.com and start_url /year. Then the rules are set for different urls. It would call year_parser for all the webpages in the format of http://www.imdb.com/year/1992 and movie_parser for webpages in the format of http://www.imdb.com/title/tt0477348/.