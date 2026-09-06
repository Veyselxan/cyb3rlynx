---
title: "Reflected XSS in a JSON POST Body (and Why I Almost Ignored It)"
date: 2026-09-06
categories: [Bug Bounty, Web Security]
tags: [xss, reflected-xss, bug-bounty,web-security]
description: "Found a reflected XSS living inside a JSON endpoint, the kind of thing I usually write off as a dead end. This one turned out to be real. Here's how I confirmed it, why the nosniff header did nothing, and the text/plain form trick I used to actually land it."
---

## The reflection I almost wrote off

Real talk: most reflections you find inside a JSON endpoint go nowhere. You throw some characters in, they bounce back, and 90% of the time the browser never treats them as anything but text. So when I catch one on a JSON route, my default reaction is a shrug, not excitement.

This one turned out to be the real deal, on a jobs platform I was poking at for a bug bounty. And honestly the payload was the boring part. The fun stuff was everything around it: proving it wasn't a dead end, figuring out *why* it was happening, and then the actual headache of delivering it when the vulnerable endpoint only wants to talk JSON.

> Target host and client name are redacted throughout. I'm walking through the class of bug and how I approached it, not handing anyone a live unfixed endpoint.
{: .prompt-info }

## The endpoint

It was a "load more" content handler taking a JSON body over POST:

```http
POST /x/x/x/loadmore HTTP/1.1
Host: target.com
Content-Type: application/json

{"xxxxlogin":"user_input_here",
 "options":{"limit":10,"sorts":[{"order":"desc","property":"publicationDate"}]},
 "page_id":5}
```

The `xxxxlogin` field looked interesting, so I did what I always do with a free-text field: threw a quick probe at it to see what the endpoint does with special characters. A marker plus a scatter of metacharacters and one benign tag, all in one shot:

```
lynx'"()<b>xss</b>
```

The marker (`lynx`) makes it easy to grep the response, the quotes and parens test whether I can break out of anything, and `<b>` is a harmless way to check if a tag survives. And the response came back with it reflected inside an error string:

```
no environment found to connect cluster error 2:super_lynx'"()<b>xss</b> cluster:
```

The `<b>` came back completely raw. That's the moment worth stopping on.

## First move: check the reflection properly

Seeing my input bounce back isn't XSS by itself. Before I get anywhere near `alert()`, I want two answers:

1. **What `Content-Type` is the response?**
2. **Does my input come back raw, or does it get HTML-encoded on the way out?**

Those two things decide whether there's even a bug. So I confirmed it cleanly with curl, watching the header and the reflection at the same time:

```bash
curl -sk 'https://target.com/x/x/x/loadmore' \
  -H 'Content-Type: application/json' \
  --data-raw '{"xxxxlogin":"lynxpoc<b>t</b>",
    "options":{"limit":10,"sorts":[{"order":"desc","property":"publicationDate"}]},
    "page_id":5}' \
  -i | grep -iE 'content-type|lynxpoc'
```

The `<b>` stays a benign tag on purpose. It can't fire anything, but if it comes back as literal `<b>` and not `&lt;b&gt;` inside an HTML response, I know the output isn't being encoded, and that's basically the whole game right there.

## Why I don't get excited about JSON reflections

Here's the way I think about these, and why I stay calm until I've actually checked.

An endpoint reflecting your input only matters if you can get the browser to **parse that reflection as HTML**. On a JSON API the default is the safe outcome:

- Response comes back `application/json` (or `text/plain`).
- Browser treats the body as data, not markup.
- Your `<script>` shows up as plain text on the page (if it shows up at all) and nothing runs.

So the usual failure mode on a JSON route is *reflected but dead*. The input bounces back, but the browser would never execute it. That right there is the vast majority of these, and it's exactly why a reflection on a JSON endpoint makes me shrug more than anything.

The one thing that flips it from dead to dangerous? A server handing back the wrong `Content-Type`. Which, well… guess what this one did.

## Okay, this one's actually real

Response headers came back like this:

```http
content-type: text/html; charset=UTF-8
x-content-type-options: nosniff
```

And the body was an error message that reflected my input with zero escaping:

```
no environment found to connect cluster error 2:super_<INPUT_HERE> cluster:
```

All three boxes for a real reflected XSS, ticked at once:

- **Response is `text/html`.** Browser parses the body as markup, so anything I inject becomes an actual DOM node.
- **Input comes back raw.** No `&lt;`, no `&gt;`, nothing. What I send is what the parser gets.
- **Barely any filtering.** I could throw mixed-case tags, quotes, ampersands, whatever, and it all sailed through untouched. If that survives, a clean payload definitely will.

This wasn't "reflected but dead." The server was straight up serving my input back as HTML. Real, exploitable, no asterisks.

## The nosniff header that does nothing

At first glance that `x-content-type-options: nosniff` looks like it's helping. It's a security header, so it must be doing *something*, right? Nope. Here it does absolutely nothing for the defender, and it's worth knowing why because people get this wrong all the time.

`nosniff` only kicks in when there's a **mismatch** between the declared type and what the browser suspects the content really is. Its whole job is to stop the browser from "sniffing", i.e. guessing that a `text/plain` response is secretly HTML and rendering it as such. It's a guardrail against the browser *overriding* the server.

There's no mismatch here to guard against. The server itself is saying `text/html`. The browser isn't guessing anything, it's just doing what it was told and rendering markup as markup. `nosniff` has no opinion about that. The actual fix was never a header. It was: stop reflecting user input into HTML, or return the correct `Content-Type` for what is obviously a data response.

> A `nosniff` header on a response that's *genuinely* `text/html` buys you exactly zero XSS protection. It stops type confusion, not markup injection. Slapping it on doesn't cover you here.
{: .prompt-warning }

## Now the annoying part: actually landing it

Confirming the bug was the easy 20%. The real work was making it *usable*, because a reflected XSS is worthless unless you can get the victim's browser to fire off the malicious request. And that request is a `Content-Type: application/json` POST, which quietly kills the two things you'd normally reach for first.

### fetch() is useless here

The instinct is a quick one-liner: attacker page does a `fetch()` to the endpoint with the payload in the body. Doesn't work, and for two separate reasons:

- `fetch()` hands you the response as a *string*. It never renders it. Even if the body is packed with live HTML, nothing parses it into the DOM, so nothing executes. Reflected XSS needs the browser to **navigate to and render** the response, not read it as text.
- Cross-origin you can't read the body anyway unless CORS lets you, and a `no-cors` request that "works" just gives you an opaque response you can't touch.

### The JSON body fights back

Fine, the clean way to force a top-level navigation cross-origin is an auto-submitting form. Except a plain form can only send three content types: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`. It **cannot** send `application/json`. Full stop.

And if you go "okay I'll just script the JSON request then", congrats, you just tripped a CORS **preflight**. `application/json` isn't a "simple" request, so the browser fires an `OPTIONS` check first, and cross-origin that check shuts you down before your real request even leaves the building.

So both doors are locked. Form can't produce a JSON `Content-Type`, scripted JSON gets preflighted into the void. This is the exact wall where people go "eh, technically reflected but not really exploitable" and move on.

I got stuck here for a bit too. The way through was noticing one thing about the server: it wasn't actually *enforcing* `application/json`. It was just reading the raw body and parsing it as JSON no matter what the header said. That little crack is all you need.

## The text/plain enctype trick

Here's the move that turns a JSON-body reflection into a zero-click cross-origin PoC.

A form with `enctype="text/plain"` sends the body almost as-is, with no percent-encoding on `<`, `>`, spaces, quotes, or braces. It joins each field as `name=value`, one per line. With a single input, the body is literally:

```
<input-name>=<input-value>
```

So I split my JSON across the input's `name` and `value` at a throwaway key, and let that literal `=` land *inside* a string value where it does no damage. The `name` carries everything up to `..."pad":"`, the `value` carries `"}`, and the `=` the browser wedges between them just becomes the value of `pad`. What the server receives snaps back into perfectly valid JSON:

```json
{"xxxxlogin":"<img src=x onerror=alert(document.domain)>", ... ,"pad":"="}
```

Because `text/plain` leaves the brackets and quotes alone, the payload arrives exactly how I wrote it. And because this is a genuine form submission, the browser does a top-level navigation to the endpoint and renders the `text/html` error page, payload and all.

## The final PoC

```html
<!doctype html>
<html>
  <body>
    <form action="https://target.com/x/x/x/loadmore"
          method="POST" enctype="text/plain">
      <input name='{"xxxxlogin":"<img src=x onerror=alert(document.domain)>",\"options":{"limit":10,"sorts":[{"order":"desc","property":"publicationDate"}]},"page_id":5,"pad":"'
             value='"}'>
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```


Open the file, form auto-submits, browser navigates, `alert(document.domain)` pops on the target's origin. No clicks, cross-origin, JSON body and all. Clean.

Two things I did on purpose:

- Went with `<img src=x onerror=...>` instead of a bare `<script>`. On a full navigation a script tag would run too, but event-handler payloads are just more reliable across contexts, so that's my default for a PoC.
- Used `alert(document.domain)` and nothing spicier. It proves *which origin* the code runs in without doing anything harmful, which is the entire point of a proof of concept. No cookie theft, no exfil. You're demonstrating execution, not building a weapon.



**Untrusted input flows into an HTML response with no output encoding.**

Following the chain:

1. Endpoint reads `xxxxlogin` from the body, fully attacker-controlled.
2. Tries to resolve it to some "environment/cluster" and fails.
3. Builds an **error message** by shoving the raw input straight into the string (`...error 2:super_<INPUT> cluster:`).
4. Returns that string as `Content-Type: text/html`.

Two mistakes stacked on top of each other. One: echoing user input into output at all without encoding it for the HTML context. Two: labeling what is clearly a machine-readable error as `text/html`, which is what invites the browser to parse the injected markup in the first place.

Error paths are a classic blind spot for this. Devs carefully validate and encode the happy path, then build error strings with lazy concatenation because "it's just an error message", forgetting the error string still contains attacker input and still gets rendered by a browser.

## How to actually fix it

Any one of these kills the bug. Do all of them for defense in depth:

- **Encode on output.** HTML-entity-encode user input before it goes into any HTML response, error messages included. This is the real fix.
- **Return the right `Content-Type`.** A data endpoint should answer `application/json`. As data, the browser won't parse injected tags and the same reflection goes inert.
- **Stop reflecting input into errors.** Log the bad value server-side, return something generic (`error 2: environment not found`). No reason to echo raw input back at all.
- **Validate input shape.** `xxxxlogin` clearly has an expected format, so reject anything with markup characters up front.

Notice what's *not* on the list: more security headers. `nosniff` was already there and doing nothing. A CSP would help contain the blast radius, sure, but that's a mitigation, not the fix.

## What I took from it

- A reflection is a lead, not a finding. On JSON endpoints it's usually a dead end, and the only thing that separates the two is whether the response is served as `text/html`. Two minutes with `curl` settles it.
- Security headers are contextual. `nosniff` protects against MIME confusion, not markup injection into a real `text/html` response. Know what a header actually does before you lean on it.
- With reflected XSS, the exploit is often the easy bit and **delivery is where the real work lives.** fetch() won't render, plain forms can't send JSON, scripted JSON gets preflighted. But a server that parses the body regardless of `Content-Type` opens the door to the text/plain trick.
- And underneath all of it? Still just unencoded input in an HTML sink, hiding on an error path. The fundamentals never stop paying off.
