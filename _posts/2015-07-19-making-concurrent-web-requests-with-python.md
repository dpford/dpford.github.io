---
layout: post
title: Making Concurrent Web Requests with Python
comments: true
---

Recently I found myself having to gather some metadata about a large number of URLs (title tag, text from page, etc.)

This is typically very easy to do in Python using Requests and Beautiful Soup:

{% highlight python %}
import requests
from bs4 import BeautifulSoup

titles = []
for url in urls:
    r = requests.get(url)
    soup = BeautifulSoup(r.content)
    title = soup.title
    titles.append((url, title))
{% endhighlight %}

That does the job for most small tasks, but what if you have to grab info on hundreds of thousands, or even millions of URLs?  By default, each link was being crawled one at a time, with the next one not starting until the previous one finished.  Why not crawl multiple links at once?  That's where concurrency comes in.

There are a couple of widely used packages for making concurrent web requests with Python: [grequests](https://github.com/kennethreitz/grequests) and [requests-futures](https://github.com/ross/requests-futures).  Grequests was made by the same guy who made [requests](http://docs.python-requests.org/en/latest/), but unfortunately looks to be abandoned as of May 2015.  Requests-futures takes advantage of the concurrency.futures package that was introduced in Python in version 3.2, but is also available as a backport in earlier versions (incluiding Python 2.7) with a simple:

 `pip install futures`

I tried both packages, but for my use case, found them to be too limiting.  Instead I started with [an example](https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor-example) from the official python docs and tailored it to my needs:

{% highlight python %}
from bs4 import BeautifulSoup
import requests
import sys
import concurrent.futures
import time

# Hopefully you'll be checking more than three URLs
URLS = ['https://www.google.com/',
        'https://github.com/',
        'https://www.facebook.com/']

#Outputting to a list for simplicity, but you'd probably want to output into some sort of datastore
output = []

# Retrieve a single page and report the url and contents
def load_url(url):
    r = requests.get(url, headers={'Connection': 'close'}, timeout=10)
    return r.content

i = 0
start = time.time()
# Using 7 workers was the sweet spot on the low-RAM host I used, but you should tweak this
with concurrent.futures.ThreadPoolExecutor(max_workers=7) as executor:
        # Start the load operations and mark each future with its URL
            future_to_url = {executor.submit(load_url, link): link for link in URLS}
            for future in concurrent.futures.as_completed(future_to_url):
                this_id = future_to_url[future][0]
                try:
                    data = future.result()
                    soup = BeautifulSoup(data)
                    if soup.title:
                        # Due to some problems with foreign titles, I encoded in ascii
                        title = soup.title.string.encode('ascii', 'ignore')
                    else:
                        title = ''
                    # This removes the styles and scripts from the soup object so I can just grab text on page
                    [s.extract() for s in soup(['style', 'script', '[document]', 'head', 'title'])]
                    content = soup.getText().encode('ascii', 'ignore')
                except Exception:
                    title = ''
                    content = ''
                i = i + 1
                # Print out things so you know what's going on behind the scenes
                sys.stdout.write("Completed link {} with title {}\r".format(i, title))
                sys.stdout.flush()
                output.append((title, content))
        runtime = time.time() - start
        sys.stdout.write("took {} seconds or {} links per second".format(runtime, 100/runtime))
{% endhighlight %}

This allows you to run the process on multiple threads at once, configurable by the `max_workers` argument passed into the  `ThreadPoolExecutor` object.

If getting all of the data is of critical importance to you, you’ll probably want to handle specific exceptions (web requests can return a number of exceptions).  For me, I wanted to get through as many URLs as possible as quickly as possible, with a reasonable (95%+) amount of accuracy, so I chose a blanket try/except.
