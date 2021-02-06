+++
date = 2021-02-05T20:20:42-06:00
tags = ["tech", "security", "privacy", "stateoftheweb"]
topics = ["security", "privacy", "stateoftheweb", "tech"]
title = 'Ditching Google Analytics for GoatCounter'
+++

### Analytics Doesn't Require Tracking
As a very fast follow to my [previous post](https://tristor.ro/blog/2021/02/05/replacing-disqus-with-commento.io/), I've now ditched Google Analytics as well.  As I noted there, I was interested in privacy-respecting alternatives, so I found quite a few interesting ones.

The options I considered were [Plausible](https://plausible.io/), [Offen](https://www.offen.dev/), and [GoatCounter](https://www.goatcounter.com/).  I ended up choosing GoatCounter because it provides a free SaaS tier for strictly personal use websites, which this is.  Plausible looks super slick, but is $4/mo even for my minimal traffic, which is almost what I pay every month for hosting.  It's just too much for a low-traffic personal website, unfortunately.

One of the things I also liked about GoatCounter was the author's writings about why they made it.  They make some very good points, and one of those is that to get basic analytics on what keywords people found your website through and which articles get the most hits, you don't need to track people.  You don't need to set a cookie.  You don't need to know anything about how that person traverses the web outside your site.  And most importantly, you don't need to share any of that data with Google or another big corp.

### Getting Rid of Google Analytics Wasn't Easy

So, sure, [hacking in GoatCounter to the theme I use](https://github.com/yoshiharuyamashita/blackburn/pull/108) was pretty straightforward, and so was disabling Google Analytics.  I had that done several hours ago.  But for some reason Google Analytics was still loading on my page.  Apparently either I, or an automated process, had enabled a Cloudflare App for Google Analytics 4 years ago, and it was injecting Google Analytics into every page.  This find was thanks to a friend of mine, Jon Kelley, helping me scour every asset imported on every page all they way down the dependency graph to figure out what was importing Google Analytics.  It's now been resolved, although the Cloudflare Apps control panel gave me several errors before successfully uninstalling GA.

### Conclusion

Anyway, this is a short post to update that I've cut Google Analytics out too, and now my site actually complies with several new regulations like GDPR and CCPA, as well as being quite a bit faster, with less external dependencies and connections, and reduced page weight.  I'm going to seek ways to improve performance further as I work through updating my site with new content and areas.