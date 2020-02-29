---
layout: post
title: Scraping Kijiji Real Estate
date: '2020-02-28'
author: Karl Engen
tags:
- Projects
- Python
- Automation
---
I was wanting to move out of the place that I was in and as such needed to find a new place to live. Rather than spending a bunch of time looking through posts I thought "Why can't I have my computer find them and send them to me."

## The Setup

First I needed to get the posts. I tried several different ideas before settling on the rss feed that Kijiji provides. This seemed to be the easiest way to get the information that I wanted. It also seemed like it might be less intrusive to kijiji's servers.

## The Requirements
1. **It must be simple.** No messing around with databases or complex algorithms.
2. **It must be portable.** Aside from the python packages it relies on, it must be able to be moved to another computer with minimal fuss.
3. **It must notify me.** I wanted something that would notify me wherever I was, at any time of the day or night.
4. **It must be low overhead**

## Building it

The first challenge I met was how to get the data that I wanted. At minimum I wanted to get the url of the post so I could look at it, but it would also be nice to get the title of the post as this is typically the address. Originally I was going to scraped the actual results page on kijiji and while this would have worked I ended up using the rss feed of the results page.

The main advantage of this is that there is less overhead. This causes less of a load on both the computer running the script and the server to which it is connected.

To begin writing the code, I first imported the packages that I would use.
{% highlight python%}
import feedparser #for parsing the rss feed
import os, shutil, re # for logging and searching the posts
from simplepush import send # for notifications
from datetime import datetime # for logging
{% endhighlight %}
At the bottom each results page, there is an rss feed link. It was this that I used to get the individual posts.
{% highlight python%}
urls=("https://www.kijiji.ca/rss-srp-for-rent/swift-current/c30349001l1700093","https://www.kijiji.ca/rss-srp-real-estate/swift-current/c34l1700093")
now=datetime.now()
print(now.strftime("%d/%m/%Y %H:%M:%S") +'--- Kijiji_Shaunavon.py')
for url in urls: # loop through urls in variable url
    NewsFeed = feedparser.parse(url) # parse url as rss
{% endhighlight %}
 The next step was to search the resulting posts for the two towns that I was potentially looking for housing in. I used both upper and lowercase search terms to help ensure that potential housing did not slip through.
 {% highlight python%}
 for post in NewsFeed.entries:
         match=re.search(r".+\bShaunavon\b.+" or r".+\bshaunavon\b.+",post.title or post.summary) or re.search(r".+\bGull Lake\b.+"or r".+\bgull lake\b.+",post.title or post.summary)
 {% endhighlight %}
If the post matched the search terms then it was deemed to be a post that I would want to see.

There would be potential to search for certain price ranges or other metadata, but I decided to just get all posts that were in my area. If someone was to do this in a larger center I could see them wanting to further restrict the results, but in my area it averages only a few posts a day.
  {% highlight python%}
  if match:
    id=post.id # set the url of post to variable
    with open('/home/karl/Scheduled Tasks/Daily/Log/Kijiji_Shaunavon_posts.log', 'a+') as notified: # open text file of previous results
        cont=[]
        contents=notified.readlines() # read contents of text file to memory
        for line in contents:
          line=line.rstrip()
          contents=cont.append(line)
  {% endhighlight %}
The contents of the log of previous posts was then compared to that of the id of the post. If the id was not found to be in the text file, the post was new and was sent to me.

Otherwise the post had already been sent and was subsequently ignored.
{% highlight python%}
                if id not in cont: # if post is not in text file, notify me
                    print(line+'-->'+id)
                    send("xJ4jv4", post.title, post.link, "Real Estate")
                    notified.write(post.id+'\n')
                    print('New Post! - '+post.id)
                else: # if post is in text file, ignore post
                    pass
{% endhighlight %}

All together the code for this project is as follows:

{% highlight python%}
  import feedparser
  import os, shutil, re
  from simplepush import send
  from datetime import datetime
  urls=("https://www.kijiji.ca/rss-srp-for-rent/swift-current/c30349001l1700093","https://www.kijiji.ca/rss-srp-real-estate/swift-current/c34l1700093")
  now=datetime.now()
  print(now.strftime("%d/%m/%Y %H:%M:%S") +'--- Kijiji_Shaunavon.py')
  for url in urls:
      NewsFeed = feedparser.parse(url)
      for post in NewsFeed.entries:
          match=re.search(r".+\bShaunavon\b.+" or r".+\bshaunavon\b.+",post.title or post.summary) or re.search(r".+\bGull Lake\b.+"or r".+\bgull lake\b.+",post.title or post.summary)
          if match:
              id=post.id
              with open('/home/karl/Scheduled Tasks/Daily/Log/Kijiji_Shaunavon_posts.log', 'a+') as notified:
                  cont=[]
                  contents=notified.readlines()
                  for line in contents:
                      line=line.rstrip()
                      contents=cont.append(line)
                  if id not in cont: #
                      print(line+'-->'+id)
                      send("xJ4jv4", post.title, post.link, "Real Estate")
                      notified.write(post.id+'\n')
                      print('New Post! - '+post.id)
                  else:
                      pass

{% endhighlight %}

## Scheduling

Now that the code was written I needed a way to run it on a schedule. For this I chose a cronjob. To make it easier to run I wrote a small shell script that would run the python script.
{% highlight bash%}
#!/bin/bash

cd /home/karl/'Scheduled Tasks'/Daily
python Kijiji_Shaunavon.py
dt=$(date '+%d/%m/%Y %H:%M:%S');
echo "$dt --- run_Shaunavon.sh ran successfully"
{% endhighlight %}

The most important thing that I found with cron is that the logging is pretty atrocious and the script can fail to run and there is no record of what went wrong. For this reason I piped the whole line into a text file called cron.log. In here I am able to get all the output of the cronjob.
  ```
*/15 * * * * /home/karl/'Scheduled Tasks'/Daily/run_Shaunavon.sh >> /home/karl/'Scheduled Tasks'/Daily/cron.log 2>&1
  ```
## Outcome

Since I wrote this bit of code it has been a great help. No longer do I have to spend time checking for new posts. While the time saved is small I do like the fact that the posts are sent to me automatically. The advantage to this is that I get them more promptly, since it isn't waiting until I have a chance to check them. As an added bonus it is a fun conversation piece.
