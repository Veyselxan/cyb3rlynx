---
title: "That Time a Bank Blocked My Account and Still Handed Me a Token"
date: 2026-06-20
categories: [Bug Bounty, Web Security]
tags: [otp, authentication, rate limiting, jwt, burp suite, broken access control]
description: A four digit OTP, a ten minute lockout, and a backend that never got the memo. The story of the laziest brute force I ever ran turning into an auth bypass.
---

This one is old. Four or five years back now, maybe more. I keep coming back to it because it is one of those bugs that had no business existing, and I almost walked right past it like an idiot.

There was a bank web app. Normal login flow, nothing fancy. You drop in your credentials, and the second you do that the server fires a four digit OTP straight to your phone. Four digits. So somewhere between 0000 and 9999. Ten thousand possible codes. The kind of number that makes a brute force lean forward in its chair.

So the first thing that pops into my head is the obvious question. Is there any rate limiting on this thing? A lockout? An attempt counter? Some kind of grown up protection sitting in front of that OTP field? You cannot tell from the outside. You can stare at the request all day and it will not whisper a single thing to you. There is only one way to find out, and that is to poke it.

So I did what any reasonable person with too much free time does. Sent the OTP verify request over to Burp Intruder, loaded up a payload list from 1000 to 9999, and let it run.

```http
POST /api/v1/auth/otp/verify HTTP/1.1
Host: redacted.bank.example
Content-Type: application/json

{"sessionId":"<redacted>","otp":1000}
```

Fifth request in, the server clears its throat and tells me my account is blocked for ten minutes. Five tries and you are out. Fine. Honestly my first reaction was not hacker brain at all, it was more like oh no. Hmmm. Okay, maybe I should not have done that. That is a real account on a real bank getting locked because I got curious. Smooth move.

So I assumed the OTP was protected, gave it a mental shrug, and went back to whatever else I had open. Closed that tab in my head. Moved on with my day.

Here is the part I love.

I never actually stopped Intruder. It was still happily chewing through the payload list in the background the whole time, completely ignored, bumping into walls like a roomba that nobody asked to keep going.

A few minutes later I glance back at the Intruder window and one row is wearing a different outfit. Status 200. Response length different from every other line in the list. Booom.

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"token":"eyJhbGci...","status":"ok"}
```

Let that sink in. The account was blocked. The message was real. The ten minute timer was real. And yet, the moment Intruder happened to land on the correct OTP, the backend validated it like nothing was wrong and handed back a perfectly valid JWT.

Think about where the protection actually lived. The lockout was standing on the front porch politely saying come back later, while the real OTP validation logic in the back never got the message. The block was basically cosmetic. The thing it was supposed to guard was still wide open and still answering the door to anyone who knocked with the right code.

I grabbed that JWT, threw it at one of the API endpoints, and it just worked. Authenticated. Real session, real access, on an account that the system itself genuinely believed was locked out at that exact moment.

Reported it to the bank, they confirmed it, and the hole got patched.

The lesson is the same one I keep relearning the hard way. A protection that looks like it is working and a protection that is actually working are two completely different animals, and the gap between them is exactly where these bugs go to live. The lockout was enforced on the wrong layer. It guarded the door instead of the lock. The correct fix is to enforce the block at the validation step itself and to stop issuing tokens at all while an account is in that locked state, not to slap a friendly error message on the response and call it secure.

And the honest part. If I had stopped Intruder the moment I saw that block message, like a normal, sensible human being, I would have walked away fully convinced the OTP was solid. The only reason I found anything at all is that I was too lazy to hit the stop button.

Sometimes the bug does not show up because you were clever. It shows up because you got distracted and left the tool running. I will take it.
