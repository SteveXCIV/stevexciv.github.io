+++
title = "Back to Blogging"
author = "Steve Troetti"
date = 2024-02-04
updated = 2024-02-04
description = "An introductory post that talks about setting up Zola."

[taxonomies]
tags = ["announcements", "zola", "tabi"]
+++

# Spring Cleaning

I was recently going through my repositories and cleaning up old stuff I knew I
probably wouldn't touch again.
It's not spring yet, but I tend to get my fill of winter somewhere around the
21st of December every year anyway, so I'll allow myself to do some early spring
cleaning.

As I was going through my list of repositories, I saw the one that backs this
site.
The last time I had touched it was when I was about as surprised as everyone
else to hear about the [untimely demise of Google Domains](https://blog.pragmaticengineer.com/google-domains-to-shut-down/).
I opted to switch providers on hearing that news, and I needed to tweak repository
settings when I was setting up the new CNAME records.
Prior to that whole incident, I hadn't touched my blog in _years_.
Suffice to say it was getting a bit dusty.

My only post on the old blog was about how I couldn't make a Jekyll theme I liked
so I just ended up choosing a default one.
Well, things are a lot different now than when I wrote that.
With GitHub supporting a full suite of CI/CD goodies and having a pretty good
free tier, Jekyll is far from the only (good) option for getting a site on GH
pages.
I never really knew much Ruby anyway, so it was right about that moment that I
decided to seek out another static site generator (SSG).

# Enter Zola

If the 🦀 crab emoji on my homepage's "about me" blurb didn't give it away, I
like Rust.
Not the reddish brown stuff that weathers ferrous items, but the programming
language that [continues to take the world by storm](https://github.blog/2023-08-30-why-rust-is-the-most-admired-language-among-developers/).
When considering what SSG I wanted to go with, it was important to me that I
found one in a language that I, a: know, and b: enjoy working with.
Rust checks both of these boxes for me, so I was pretty happy when I came across
[Zola](https://getzola.org).
Zola is written in Rust, which to me means that I can troubleshoot it more easily,
and I could potentially contribute at some point down the line.

I looked around a bit and also ended up finding the excellent theme [tabi](https://github.com/welpo/tabi).
It's loaded with configuration to customize my blog exactly how I please,
it's secure by default, and the project's [own GH pages](https://welpo.github.io/tabi/)
uses tabi (of course), and serves as project documentation - _neat_.

Zola also has excellent documentation for getting set up on GH Pages, and there's
a [custom GH Action](https://github.com/shalzz/zola-deploy-action) for deploying.

Overall, I'm quite happy with how this turned out.
So maybe this time around I'll actually take the time to write about some of
the things I'm working on in my personal time 😅.

All-in-all, it's been a good weekend doing some spring cleaning, and now my blog
is powered by Rust, and that's always a plus.
