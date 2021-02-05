+++
date = 2021-02-05T15:38:39-06:00
tags = ["tech", "security", "privacy", "stateoftheweb"]
topics = ["security", "privacy", "stateoftheweb", "tech"]
title = "Replacing Disqus with Commento.io"
+++


### If You're Not The Customer, Then You're The Product

You've probably heard this statement before, and I don't know that it's always true, but it's become something of an axiom in the web/internet space.  It's true enough in the ways that matter, though, and that brings us to the topic of the day.

Today a post made it to the front page of [HackerNews](https://news.ycombinator.com/item?id=26033052) written by Supun Kavinda on his blog entitled ["Disqus, the dark commenting system"](https://supunkavinda.blog/disqus).  Thanks to the comments on HN about this post, I found out that I had somehow missed an announcement that Disqus was acquired by an ad-tech company in 2017, probably while I was still traveling and actively updating this site.  I was also apalled by how user-hostile the tracking behavior of Disqus is.

### Replacing Disqus with Commento.io

If you read my weblog (which you probably don't, I don't get many visitors), you've already noticed that Disqus is gone.  Thanks to some other comments in the HN thread, I was able to quickly come up with a list of possible alternatives, two of which I zeroed in on as likely choices: [Isso](https://posativ.org/isso/) and [Commento.io](https://commento.io/).  After a quick analysis of their features and construction, I determined that self-hosting the comments wasn't in line with the low effort I wanted to put into administrating the backend for my blog, and that I was willing to pay actual money to get the feature, even though I know almost nobody will use it.  Both Isso and Commento support importing from Disqus exports, and seemed simple enough to setup, but the SaaS option from Commento seemed good and was pretty easy to [hack into the Hugo theme I'm using](https://github.com/Tristor/blackburn/tree/commento).

So, with that, I chose Commentio.io.  I paid my $99 for the yearly plan after verifying I could get it working and that the import of Disqus comments was successful.  The original author of the Blackburn theme hasn't been responding to pull requests, so I am now running my theme off my fork, as it diverges with two patches (and likely growing).  Thankfully Commento is very well built and super lightweight, it actually made the page weight and page load time go down compared to Disqus, and my site was already blazing fast and minimalistic.  I was able to get everything set up and hacked in within about 20 minutes of effort, most of it messing with the theme and testing since it'd been awhile since I'd messed with Hugo partials.

I am going to try to start keeping things up on this site and add additional sections, so this is a good step to take to bring this back from the dead.  As a way of putting my money where my mouth is when it comes to online privacy, I won't be subjecting my visitors and users to things I find objectionable myself.  I'm now actively looking for alternatives to Google Analytics, which this site currently uses as well.  I have never had advertising on this site, and I never will have advertising on this site, but it is nice to know if anyone reads it.  If you are aware of solid analytics packages which I could self-host or which make meaningful privacy commitments, I'm all ears.  I'm considering setting up one of the classics that just parses server logs.