---
title: 'A Cool Pattern for Ingesting Emails for LLM Extraction Jobs'
description: 'Cloudflare Email Workers to the Rescue!'
pubDate: 2026-08-05
tags: [ 'ai', 'llm', 'ai pipelines', 'cloudflare' ]
draft: false
---

I love the idea of local LLMs, but I don't really have the GPU to justify doing much on them. I've got 64 gigs of CPU
RAM on an old 24/7 machine on my network, but I'm not getting much high-context, fast-feedback coding assistance from
it. It's a bummer, because I like playing around with this stuff to see what it can handle and what it can't and there
are certainly a lot of limits to what I can do. But there's one thing I can do well - background jobs. And I love
background jobs.

One of the best ways to utilize that, IMO, is periodic event streams - RSS feeds or emails. I get a lot of emails about
music and a lot of emails about events happening in Nashville - a perfect use case for parsing well, but slowly.

`qwen3:30b-a3b` works pretty well CPU only. I get about 12 tokens per second, and it's a big enough model that it can
handle pretty much anything. But how do I get those emails to that model? I don't love the idea of giving a model
unfettered access to my Gmail account, so what's the better option?

## Enter Cloudflare Email Workers

Being a dev, I own several domains. It's part of the gig. But I don't want to host my own email server - it's
complicated and overhead I don't want to deal with. So what I've done in the past is forward email using Cloudflare.
It's easy enough to create forwarding rules: me@mydomain.com -> my Gmail account. I've set up labels in Gmail to say if
it comes *to* a particular address, it gets treated differently once it hits my inbox. A throwaway @ gets me signups for
promotions without risking my address getting sold. I even created a lightweight worker once to forward emails to
multiple accounts - dumb school signups that only have one email input, but should go to both Lindsey and me. But now
I've found an even better Worker setup, and it fits perfectly with the LLM pipelines I'm trying to build, and it's super
simple.

Basically, I'm prepping for 2 lanes: events and music. Each of those has a distinct email address, and both are routed
to my worker. Then it's just a few lines of code:

```ts
export default {
    async email(message, env, _) {
        const prefix = message.to.split('@')[0];

        const raw = await new Response(message.raw).arrayBuffer();
        const key = `${prefix}/${message.to}/${Date.now()}-${crypto.randomUUID()}.eml`;

        await env.INGEST_BUCKET.put(key, raw, {
            customMetadata: {
                to: message.to,
                from: message.from,
                prefix
            }
        });

        console.log('stored', {prefix, key, bytes: raw.byteLength});
    }
} satisfies ExportedHandler<Env>;
```

Now these specific emails are in R2 Object Storage, which has a generous free tier. And they're ready for extraction and
use in any sort of processing pipeline! Simple, free, and easy enough for anyone to do. Plus it keeps your real inbox
clean. Highly recommended.

Next up: how do I use local AI to extract, parse and even enrich these payloads? It'll be a good one. 
