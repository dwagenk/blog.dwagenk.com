---
layout: post
title: Linking this blog with my mastodon profile
category: fediverse
tags: mastodon, fediverse, 15WeeksToBlog
excerpt_separator: <!--more-->
---

Alongside the revival of this blog through yesterdays post I also added a link
to my mastodon profile in its footer. My mastodon profile also had a link to
this blog in it for a long time, though it missed the little "verified"
checkmark next to it. In case you've not seen the mark I'm referring to or are
not present on mastodon / the fediverse yet (you should join, e.g.
[here](https://fosstodon.org) or [here](https://social.tchncs.de)), you can see
it in this screenshot: 

![Screenshots of verified blog address in tusky android
app](/assets/2020-04-30-linking-mastodon-verified-screenshot.png)

Getting it in there isn't complicated and there's several [online
resources](https://docs.joinmastodon.org/user/profile/) on achieving that.
Basically it's just adding a URL in one of the mastodon profile metadata fields
and then linking back to your mastodon profile on that website with a
`rel="me"` attribute.

```html
<a href="https://chaos.social/@dwagenk" rel="me">My Mastodon Profile</a>
```

One thing I didn't see mentioned anywhere: The link verification doesn't appear
immediately, but needs some time. Apparently the mastodon instance server scans
those URLs periodically instead of every time the profile is viewed (sounds
like a reasonable design decision favoring lower load times). This irritated
me, because it needed more than 9 hours for the checkmark to appear on my
profile. The timespan got me thinking and double checking, whether I made any
mistakes linking back, but in the end waiting was just the best solution for
this problem...

<!--more-->
----

This post is part of the
[#15WeeksToBlog](https://chaos.social/web/timelines/tag/15WeeksToBlog)
challenge.<br/> Week 1/15, 2nd post this week.

Feel free to contact me via [mail](mailto:dwagenk@mailbox.org) or
[mastodon](https://chaos.social/@dwagenk) if you've got any notes on this post
or start a public discussion via this blogs [github issue
tracker](https://github.com/dwagenk/blog.dwagenk.com/issues). You can also
subscribe to the [RSS feed](/feed.xml) to stay tuned! 

