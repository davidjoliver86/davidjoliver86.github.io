---
title: "Part 1/4 - Create and deploy an automated, serverless Newegg stock checker using Python, Lambda, and Terraform"
date: 2020-11-29
draft: false
tags:
  - aws
  - python
---
> ⚠️ This tutorial is a relic of its time period, back when the COVID-19 pandemic was
  beginning to surge. While the information is still relevant and factual, this tool
  was designed with once-in-a-lifetime supply chain conditions in mind, and the end
  result may not be as useful today as it was back when this article was written.

In this series, we're going to create a "stock checker" in Python, and then later on,
we're going to deploy it on AWS. We're not going to spin up any EC2 instances either;
this is going to be running as a "serverless" bot. And we're not going to need to have
to do much, if any, clicking and pointing in the AWS console, thanks to the power of
Terraform.

### Stock checker? But Newegg's a privately owned company...

To clarify, by "stock checker", we're going to be writing a bot to check whether certain
_products_ are in stock. I'm a gamer, and as my current gaming rig is beginning to show
its age, I'm eyeballing components for a new build. With the recent releases of the
Nvidia GTX 30 series, the Radeon 6800 series, not to mention the PS5 and Xbox Series X,
all under the murky clouds of the COVID pandemic, demand for these products is red hot.
And unless you want to deal with ridiculuous markups from scalpers on eBay, your best
bet is to find out when you can swoop in and grab an item while it's in stock.

## Building the Scraper

Anyone who's dabbled in the Python ecosystem should be aware of many quality libraries
that suit this use case - `requests` for dealing with HTTP requests, and `BeautifulSoup`
to parse and extract HTML. But as part of a self-imposed challenge, as well as a way to
make our deployment in the next post a little simpler, we're going to be sticking with
the standard library. So we'll be using `urllib3` to deal with the HTTP request, and
`html.parser.HTMLParser` to parse the response.

To view the source code as it stands for part 1, please visit https://github.com/davidjoliver86/newegg-scraper/blob/local/newegg.py.
In the example we're working with, we'll be looking for my target graphics card - the
GTX 3070.

## Using HTMLParser

We'll be following the recommendations of the official documentation: https://docs.python.org/3/library/html.parser.html:

> An HTMLParser instance is fed HTML data and calls handler methods when start tags, end tags, text, comments, and other markup elements are encountered. The user should subclass HTMLParser and override its methods to implement the desired behavior.

There are many different handler methods, but the only ones we're concerned about are:
* `handle_starttag(self, tag, attrs)`
* `handle_endtag(self, tag)`
* `handle_data(self, data)`

Let's take a look at how they're implemented in our scraper:
```python
def handle_starttag(self, tag, attrs):
    # Always increment depth.
    self._depth += 1

    # Did we hit an item?
    if tag == 'div' and ('class', 'item-cell') in attrs:
        self._encountered_item = True
        self._item_depth = self._depth

    # Are we in an item-cell, and is this the element of the item's title?
    if self._encountered_item and tag == 'a' and ('class', 'item-title') in attrs:
        self._encountered_title = True

def handle_data(self, data):
    # If we find an item's title within its cell, add it to the items dict. Assume
    # it's in stock until (very shortly later) proven otherwise.
    if self._encountered_title and self._encountered_item:
        self._title = data
        self.items[data] = True
        LOGGER.debug("Found product: %s", data)
    if data == 'OUT OF STOCK':
        self.items[self._title] = False
        LOGGER.debug("Out of stock : %s", self._title)

def handle_endtag(self, tag):
    # Always decrement depth.
    self._depth -= 1

    # Always assume we're out of a title element.
    self._encountered_title = False

    # Did we exit an item-cell?
    if self._encountered_item and self._depth == self._item_depth:
        self._encountered_item = False
```

### How deep can we go?
What isn't visible in above excerpt are some attributes that we've defined in our
`HTMLParser` subclass. Particularly, we're tracking the depth of how many tags we're
nested under. As it turns out, an `item-cell` element also has its own complex nesting
of tags, and so by keeping track of our current "tag depth", we can't assume we've
exited an `item-cell` just cause we've hit an end tag.

### The great encounters
The main way we know whether we've hit our target elements - either a
`<div class="item-cell">` element, or a `<a class="item-title">` element, is through
the `handle_starttag` method. This method returns a 2-tuple: the tag itself, and a list
of 2-tuple `('attribute_name', 'attribute_value')` pairs. We then set the `encountered`
flags accordingly so the parser knows whether it's in an `item-cell` element, and
particularly an `item-title` element within the `item-cell`.

### Domo Arigato, Mister Databoto
Once we've hit an `item-title` - that is, both `self._encountered_item` and
`self._encountered_title` are True, then we can pull out the title of the product. For
now, at that immediate moment in time, we assume the item will be in stock, and we store
it as such in the `items` dict. However, as we go to the very next element, we will
quickly be proven wrong.

Just a few elements under the `item-title` - if an item is indeed out of stock, there'll
be a small element simply saying "OUT OF STOCK". So once we hit that element, we know
to go back to mark that product as False - or "out of stock". Because the parser works
strictly in one direction, we can assume we're within the same item block, otherwise
the subsequent call to `handle_endtag` would've marked that False, meaning the next time
that flag gets flipped, it's a different product.

### The rest
The rest of this simply covers formalities with running the script interactively -
starting the parsing, then outputting the result.
```python
    def fetch(self):
        """
        Fetch data from provided URL and start parsing.
        """
        with urllib.request.urlopen(self._base_url) as f:
            self.feed(f.read().decode('utf-8'))
    
    def get_items_in_stock(self) -> List[str]:
        """
        After parsing the page, return a list of items in stock.
        """
        return [item for item, avail in self.items.items() if avail]


if __name__ == "__main__":
    # Setup console logging if running interactively.
    LOGGER.addHandler(logging.StreamHandler())

    # Do the thing.
    parser = NeweggParser('https://www.newegg.com/p/pl?d=gtx+3070&N=100007709&isdeptsrh=1&PageSize=96')
    parser.fetch()
    print(parser.get_items_in_stock())
```
The final output, as expected, is sadly an empty list, but the logging confirms that
we're hitting the products correctly. We'll be making some tweaks to this script in
a future post to get this ready to deploy to AWS Lambda, and then we can hopefully
relax for a bit until I get the message that there's some stock available.