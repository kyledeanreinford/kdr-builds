---
title: 'Building This Blog On My k3s Cluster'
description: 'Going from Zero to Deployed.'
pubDate: 2026-04-02
tags: [ 'claude', 'ai', 'tech', 'software', 'personal', 'k3s' ]
draft: false
---

Welcome to my new blog. If you're here, you probably know me. If not, [here's a recap](https://kyledeanreinford.com/blog/how-do-i-begin/) of who I
am. This blog is part musings, part showcase, part just fun. Wanna hear about my Detroit style sourdough recipe? How
about my thoughts on cosmic insignificance? Or why I use IngressRoutes instead of Ingress for my cluster? Done.

I needed an outlet, and the NY Times said no.

## Why Self-Host a Blog?
I'm a software dev with a homelab. I can throw a blog out there quickly and easily. I've used Substack for other
writing, but for a variety of reasons this felt like something I wanted to roll myself.
The fun part about building it yourself is that you get to understand all the pieces of the puzzle - one of my favorite
parts about building my own house. (Think you know the challenges of fullstack development? Try personally renovating a
house from the 1940s while living in it and connecting all the electrical and plumbing while a pandemic is going on).
Plus when something breaks there's only one person you can blame...

## The Cluster
At home I run a k3s server. It's made up of three nodes - a couple of old Meerkats and a desktop build from 12 years
ago. They're always on and run a variety of services for me - everything from home automation to a Minecraft world my
son and I can build together.
The reason for it in the first place? I was running a few services on an old Macbook and apps would fail or timeout or
just struggle one way or another. It wasn't robust enough for what I needed it to do.
Then along came a client that wanted to build a Kubernetes server with multiple environments and a bunch of automation.
I knew some things about K8s, but not enough to really be able to sink my teeth in at the enterprise level, so I read
some books (thanks Nigel Poulton!) and spun one up at home. The amount I have learned about scalability and data sharing
and networking has been invaluable - and my family benefits as well.
The whole thing is served up on Cloudflare. Secure, but accessible, and easy to debug even when I'm away from home
thanks to [Tailscale](https://tailscale.com/) - incredible product, if you've not heard of it.

## Why Astro
I wanted something dead simple for this blog. I didn't want to do the thing where deciding on the tool was more work
than actually implementing it. This is an extremely low-risk decision, so just getting something simple out there
quickly and then iterating is really the way I like to work.
Astro is a name I've heard a few times over the years - mostly when playing around with Cloudflare and their product
suite. It's fast, simple and there's no server-side runtime. I can just push a Markdown file and there's my blog.
There were a few other options I explored, including Substack, Gatsby and Hugo, but Astro checked all of the boxes. They
were recently acquired by Cloudflare as well. I generally have a very high opinion of the things Cloudflare does - if
they like it, I probably want to use it.

## How I Actually Built It
I [used Claude to build](https://kyledeanreinford.com/blog/falling-for-claude/) this site. The idea of the blog itself came from
conversations I've had with Claude about career progression and feeling like I need a creative outlet somewhere, so it
made perfect sense to 1) come up with a plan in Claude.ai, 2) have it save the plan to AFFiNE (my self-hosted
Notion-like app) where 3) Claude Code could pick up the context and start writing ASAP. All in all it took me 18 prompts
from idea -> deployed blog with all the k3s manifests, CI/CD in place as well as a mailing list and user tracking. And a
few of those prompts were just clarification questions from my end.
How does my code move from idea to living, deployed reality? I have a few levers in place for this to get it up quickly
to k3s. I've got a Github Workflow so that every time I push a blog post, my tests run on CI, and if they pass they get
pushed to a registry - I self-host Harbor for this. Github's UX around personal repo secrets is pretty annoying - you
have to go in and set them for each repository, since there are no global secrets (only for Organizations). I built a
fun little script to solve that in my terminal:

```bash
harbor-secrets() {
local repo="${1:?Usage: harbor-secrets owner/repo}"
gh secret set HARBOR_REGISTRY --body "$(op read 'op://Shared/GHAHarbor/registry')" --repo "$repo"
gh secret set HARBOR_ROBOT_USERNAME --body "$(op read 'op://Shared/GHAHarbor/username')" --repo "$repo"
gh secret set HARBOR_ROBOT_TOKEN --body "$(op read 'op://Shared/GHAHarbor/token')" --repo "$repo"
}
```

I prefer an iterative approach - ship early, then improve on it over time. My current setup can go from me having an
idea to having something cleanly deployed in 30 minutes or less. And each iteration takes about 90 seconds from pushing
changes to viewing them live. You can probably tell why it's so tempting to spend "just 5 more minutes" and get sucked
into the cycle.

## Closing
I love how it turned out. I have a basic, but functional, blog that's easy to update and will look even better in the
future.

What's next? Maybe a post-mortem on a Long Covid tracking app I built for my wife. Or something about whether or not
writing software with AI is still meaningful. Or even Boredom Therapy. Stay tuned!