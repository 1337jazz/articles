# Middleware in NextJS done right

---

TL;DR: Go to the end to get code you can copy/paste in to your proxy.ts or middleware.ts file to make your middleware actually work like middleware, and your life easier.

---

## Table of Contents
- [The Problem](#the-problem)
- [The Naive Solution](#the-naive-solution)
- [The Real Problem: Lost State](#the-real-problem-lost-state)
- [The Solution: Chain of Responsibility](#the-solution-chain-of-responsibility)
- [Implementation](#implementation)
- [Usage](#usage)
- [Complete Code](#complete-code)

---

Hello, more NextJS fuckery from me. I just spent a day coming to the conclusion that our favourite SSR framework, like with logging, is poo poo at middleware.

## The Problem

You might know that if you want to use middleware (now called "proxy") in NextJS you have to create a middleware.ts or proxy.ts file in the root of your project and do something like this:

```typescript
export const proxy = async (req: NextRequest) => {
  const token = req.headers.get('token');
  if(req.nextUrl.pathname.startsWith('/auth')) {
    user = await getUserByToken(token);

    if(!user) {
      return NextResponse.redirect('/login');
    }

    return NextResponse.next();
  }
}
export const config = {
  matcher: ['/((?!_next/|_static|_vercel|[\\w-]+\\.\\w+).*)'],
};
```

This is a pretty simple example of protecting routes that start with /auth. It works fine, but what happens when you have more logic that you need to apply to each request? For example:

```typescript
export const middleware = async (req: NextRequest) => {
  let user = undefined;
  let team = undefined;
  const token = req.headers.get('token');

  if(req.nextUrl.pathname.startsWith('/auth')) {
    user = await getUserByToken(token);

    if(!user) {
      return NextResponse.redirect('/login');
    }

    return NextResponse.next();
  }

  if(req.nextUrl.pathname.startsWith('/team/') || req.nextUrl.pathname.startsWith('/t/')) {
    user = await getUserByToken(token);

    if(!user) {
      return NextResponse.redirect('/login');
    }

    const slug = req.nextUrl.query.slug;
    team = await getTeamBySlug(slug);

    if(!team) {
      return NextResponse.redirect('/');
    }

    return NextResponse.next();
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/|_static|_vercel|[\\w-]+\\.\\w+).*)'],
};
```


What if wanted to now add:
- Logging
- Rate limiting
- Analytics
- Internationalization
- Custom headers

Hopefully you can see how this could quickly become a disgusting 5000 line function after your app is a few years old, it just doesn't scale, and it's easy to screw up the logic.

## The Naive Solution

At this point you might be screaming "ACKCHULLY you can just break out the logic into separate functions and call them in the middleware!". Okay, sure, let's take another example:

```typescript
export function proxy(request: NextRequest) {
  const securityResult = addSecurityHeaders(request);  
  const corsResult = corsMiddleware(request);          
  return corsResult; 
}
```

Okay, so this looks fine, the proxy function is like 5 lines now! Problem fixed, right? 

## The Real Problem: Lost State

Unfortunately not... Let's look a little closer at these two functions:

```typescript

const addSecurityHeaders = async (req: NextRequest) => {
  // We want to add security headers that subsequent middlewares can see
  const response = NextResponse.next();
  response.headers.set('X-Frame-Options', 'DENY');
  return response; 
}

const corsMiddleware = async (req: NextRequest) => {
  // We want to add CORS headers to the SAME response
  const response = NextResponse.next();
  response.headers.set('Access-Control-Allow-Origin', '*');
  return response; 
}

```

Can you see the problem? Each middleware function is creating a new response object, so the headers set in addSecurityHeaders are lost when corsMiddleware creates a new response object. This means that only the headers set in the last middleware function will be present in the final response:

```typescript

export function proxy(request: NextRequest) {
  const securityResult = addSecurityHeaders(request);  // Okay so now the response has security headers
  const corsResult = corsMiddleware(request);          // Creates fresh response, now with CORS headers only
  return corsResult;                                   // Only CORS headers are present in the final response
}
```
And remember, we can't just change the function signatures to accept and return a response object, because NextJS expects the middleware/proxy function to have the signature (req: NextRequest) => NextResponse. Your only option if you want to make this work is to either go back to the giant function, or do some really gross mutation of a mutable shared response object. That's awful practice and you're going to land yourself in some really hard to debug situations down the line, as well as wasting a bunch of memory and CPU cycles creating multiple response objects.

## The Solution: Chain of Responsibility

So, how do we fix this? The answer is to use the so-called "chain of responsibility" pattern, which is a fancy way of saying "how EVERY OTHER FUCKING WEB FRAMEWORK does middleware".

We need a way to *chain* middleware functions together, passing the response object from one to the next, maintaining the mutations. For example you might be familiar with this pattern from ExpressJS:

```typescript
app.use((req, res, next) => {
  res.setHeader('X-Frame-Options', 'DENY');
  next();
});

app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', '*');
  next();
});
```

or even more succinctly, if you move the logic into separate functions:

```typescript
app.use(addSecurityHeaders);
app.use(corsMiddleware);
```

## Implementation

To achieve this in NextJS, we can create a simple (although kind of tricky to get your head around) middleware chaining function. Here's an example implementation:

First just some type aliases for clarity:

```typescript
// This is just a type alias for a middleware function, we expect it to take a request and return a promise that resolves to a response
type Middleware = (req: NextRequest) => Promise<NextResponse>; 

// This is a (technically "higher-order") function that takes a middleware and returns a new middleware
type MiddlewareWrapper = (next: Middleware) => Middleware; 
```

Then we want something that can chain these middlewares together, wrapping them so that each one calls the next:

```typescript

function chain(wrappers: MiddlewareWrapper[]): Middleware {
  // The final middleware simply calls NextResponse.next()
  const finalMiddleware: Middleware = () => NextResponse.next();

  return wrappers.reduceRight((currentMiddleware, wrapper) => wrapper(currentMiddleware), finalMiddleware);
}
```

This function kind of made my brain hurt, but it's basically looping through the wrappers in REVERSE order, by first using the "final middleware" as the initial value, so it ends up being the innermost call.
 
E.g. if we have wrappers = [A, B, C], we loop backwards so C is wrappers[2], B is wrappers[1], A is wrappers[0]:

Start:    currentMiddleware = finalMiddleware
i=2 (C):  currentMiddleware = C(finalMiddleware)
i=1 (B):  currentMiddleware = B(C(finalMiddleware))
i=0 (A):  currentMiddleware = A(B(C(finalMiddleware)))

Result: A(B(C(finalMiddleware)))

When a request arrives, it flows through in correct order:
Request → A → B → C → finalMiddleware → Response

If we did it in normal order, we'd end up with:
A(finalMiddleware) → B(A(finalMiddleware)) → C(B(A(finalMiddleware)))

Yes, I hate it too, but this is the only reasonable way to do it in Next.js without more fuckery.

## Usage

But it means that we can finally just have:

```typescript
function addSecurityHeaders(next:Middleware): Middleware {
  return async function (request: NextRequest) {
      const response = await next(request);
      response.headers.set('X-Frame-Options', 'DENY');
      return response;
  }
};

function corsMiddleware(next:Middleware): Middleware {
  return async function (request: NextRequest) {
      const response = await next(request);
      response.headers.set('Access-Control-Allow-Origin', '*');
      return response;
  }
};
```
And then finally in our middleware.ts or proxy.ts file:

```typescript
const middlewareChain = chain([
  addSecurityHeaders,
  corsMiddleware,
  // Add more middleware wrappers here
]);

export function proxy(request: NextRequest) {
  return middlewareChain(request);
}
```

## Complete Code

There! Now all of your middleware functions can operate on the same response object, are separated for clarity, and you can easily add more middleware and see the order instantly! it's a little more complex to set up, but really this is the full code without all my spam:

```typescript
import { NextRequest, NextResponse } from 'next/server';
type Middleware = (req: NextRequest) => Promise<NextResponse>; 
type MiddlewareWrapper = (next: Middleware) => Middleware;

function chain(wrappers: MiddlewareWrapper[]): Middleware {
  const finalMiddleware: Middleware = () => NextResponse.next();
  return wrappers.reduceRight((currentMiddleware, wrapper) => wrapper(currentMiddleware), finalMiddleware);
}

// Example 1: Middleware that doesn't modify the response
function logRequest(next:Middleware): Middleware {
  return async function (request: NextRequest) {
    console.log(`Request: ${request.method} ${request.url}`);
    return next(request);
  }
};

// Example 2: Middleware that modifies the response
function addCustomHeader(next:Middleware): Middleware {
  return async function (request: NextRequest) {
    const response = await next(request);
    response.headers.set('X-Custom-Header', 'MyValue');
    return response;
  }
};

const middlewareChain = chain([
  logRequest,
  addCustomHeader,
  // Add more middleware wrappers here
]);

export function proxy(request: NextRequest) {
  return middlewareChain(request);
}
```

