# Logging in NextJS: Another Vercel Lock-in Masterclass

Calling all NextJS enthusiasts...

I found [this article](https://www.tomups.com/posts/log-nextjs-request-response-as-json/) after trying to figure out why I couldn't see requests being logged in NextJS. Luckily I wasn't crazy and Vercel does that on purpose. This guy talks about the lengths one has to go to just in order to implement request/response logging in NextJS.

## TL;DR

When you run `npm run dev` during development you get some information logged to stdout about requests made to your server, those lines like `GET /about 200 in 158ms` etc. That's pretty expected since every web server I've ever worked with has some default request/response logging. In production you would expect this same information, enriched with more context, in a structured format (usually JSON), which you can decide to log to the usual suspects: stdout, files, database, datadog etc for later analysis/monitoring. 

What you actually get is: **jack shit**.

## The Problem

NextJS doesn't give you a built in structured logger at all in fact - bizarre for a web framework until you realise that if you deploy to Vercel you *do* get structured logging, and request/response logging in production. So clearly this was a decision taken with the intention to get more customers, which, hey, it's a free framework, get that bag I guess.

## The Workaround

The [article linked above](https://www.tomups.com/posts/log-nextjs-request-response-as-json/) basically describes how to hack pino into the node_modules NextJS server package and keep it consistently applied between updates/installs. Then you can use pino as usual for structured, severity-based logging. 

The next challenge is how to get any kind of correlation or trace ID - unsurprisingly you get this for free as well when you deploy with daddy Vercel.
