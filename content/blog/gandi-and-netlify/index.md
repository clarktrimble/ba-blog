---
title: Gandi and Netlify
description: The domain name system is your friend.
date: 2023-09-26
tags:
  - blog
layout: layouts/post.njk
---

This is a quick post to memorialize setting up Gandi and Netlify so that Ba Blog is available via a catchy domain name.

Disclaimer: `clarktrimble.online` and `profound-gumdrop-ae4dd4` are specific to Ba Blog.
Of course, you'll use your own domain and Netlify names if you're following along from home.

## Gandi

I chose Gandi for DNS after a small scourge of the interwebs.
Mostly good to me so far.
My first order was cancelled with an email mumbling about a copy of my ID or passport, but I suspect this was because I forgot to confirm my email address.
Second time was a charm and `clarktrimble.online` is all mine for $3 US!

In the meantime, the blog is coming to life at `profound-gumdrop-ae4dd4.netlify.app`.
So, to tell Gandi to resolve the name to its address at Netlify:


{% image "./gandi-dnsrecords.png", "Gandi DNS Records" %}

The lines of interest:

```text
@ 1800 IN ALIAS profound-gumdrop-ae4dd4.netlify.app.
www 10800 IN CNAME profound-gumdrop-ae4dd4.netlify.app.
```

Important: trailing periods are _necessary_ and easy to lose when cutting and pasting!

The ALIAS record will conflict with an A record (Gandi default) that needs to be deleted.
(You can just add the two lines and Gandi will tell you which record is in conflict.)

In the new ALIAS record:
 - `@` refers the root of the domain, `clarktrimble.online` in this case
 - `ALIAS` directs the lookup to instead lookup a different name, `profound-gumdrop-ae4dd4.netlify.app` in this case

The `www` record does something similar for `www.clarktrimble.online` and probably can be left out.

Clicking through and the changes are live, nice!

## Trying It Out

Once DNS has had a few minutes to propagate, `clarktrimble.online` in a browser yields a 404 from Netlify.
Making some progress here!

And maybe a glimmer of a learning opportunity at this juncture.  Digging (har) a little:

```bash
$ dig @8.8.8.8 clarktrimble.online

; <<>> DiG 9.18.16-1~deb12u1-Debian <<>> @8.8.8.8 clarktrimble.online
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1025
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;clarktrimble.online.           IN      A

;; ANSWER SECTION:
clarktrimble.online.    20      IN      A       54.161.234.33
clarktrimble.online.    20      IN      A       44.217.161.11

;; Query time: 192 msec
;; SERVER: 8.8.8.8#53(8.8.8.8) (UDP)
;; WHEN: Tue Sep 26 09:41:03 CDT 2023
;; MSG SIZE  rcvd: 80
```

Then curl the IP with a host header:

```bash
$ curl -vL -H "Host: profound-gumdrop-ae4dd4.netlify.app" 54.161.234.33
*   Trying 54.161.234.33:80...
* Connected to 54.161.234.33 (54.161.234.33) port 80 (#0)
> GET / HTTP/1.1
> Host: profound-gumdrop-ae4dd4.netlify.app
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 301 Moved Permanently
< Content-Type: text/plain; charset=utf-8
< Date: Tue, 26 Sep 2023 14:41:45 GMT
< Location: https://profound-gumdrop-ae4dd4.netlify.app/
< Server: Netlify
< X-Nf-Request-Id: 01HB8Z0QE82WPYRBBXW0DXY1VM
< Content-Length: 59
<
* Ignoring the response-body
* Connection #0 to host 54.161.234.33 left intact
* Clear auth, redirects to port from 80 to 443
* Issue another request to this URL: 'https://profound-gumdrop-ae4dd4.netlify.app/'
*   Trying 44.219.53.183:443...
* Connected to profound-gumdrop-ae4dd4.netlify.app (44.219.53.183) port 443 (#1)
* ALPN: offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
...
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
< HTTP/2 200
< accept-ranges: bytes
< age: 0
< cache-control: public,max-age=0,must-revalidate
< content-type: text/html; charset=UTF-8
< date: Tue, 26 Sep 2023 14:41:45 GMT
< etag: "56a0f00dae9fae9f8b008bf02bfe6853-ssl"
< link: <https://clarktrimble.online/>; rel="canonical"
< server: Netlify
< strict-transport-security: max-age=31536000; includeSubDomains; preload
< x-nf-request-id: 01HB8Z0QMVCN3PWVWV9Z7EQWNN
< content-length: 10184
<
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ba Blog</title>
...
```

Soo, if we curl the resolved IP with an appropriate host-header it works.

## Netlify

You've heard of them?  So cool -- fingers crossed they survive monetization.

Back to our story: it make sense that Netlify's IP isn't just serving the lowly Ba Blog.
How does it know?  Or not, in the case of trying to get there via browser.
From the curl above, we see the host header is the magic.  Poking at the UI:

{% image "./netlify-domain.png", "Netlify Domain" %}

we're able to tell Netlify about `clarktrimble.online` and voila, clarktrimble is online : )

## SSL Postscript

Between Gandi and Netlify, SSL is just working, bravo!
