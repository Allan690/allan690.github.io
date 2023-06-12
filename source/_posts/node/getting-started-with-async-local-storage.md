---
title: Getting Started with Async Local Storage API In Node.js
date: 2020-05-20 08:15:00
categories:
- [web]
tags:
- nodejs
---

## Introduction

 <a rel="nofollow noopener" target="_blank" href="https://nodejs.org/api/documentation.html">Node.js version 14</a>, shipped with some interesting experimental additions to the core API, among which was the <a rel="nofollow noopener" target="_blank" href="https://nodejs.org/api/async_hooks.html#async_hooks_class_asynclocalstorage">async local storage API </a>. This API provides a native implementation of web request handling akin to <a rel="nofollow noopener" target="_blank" href="https://www.c-sharpcorner.com/UploadFile/1d42da/working-with-thread-local-storagetls-in-C-Sharp/">thread-local storage </a> in multi-threaded languages. Before the addition of this API, users who wanted to leverage this capability in Node.js, mostly turned to the <a rel="nofollow noopener" target="_blank" href="https://github.com/Jeff-Lewis/cls-hooked">cls-hooked library </a>, which beneath the hood is implemented using <a rel="nofollow noopener" target="_blank" href="https://nodejs.org/api/async_hooks.html#async_hooks_async_hooks">async hooks </a>.

I actually came across the async local storage API while reading up the official documentation on async hooks -  a module in Node.js that provides an API to track async resources in a Node application. Async resources inherit from the async wrap class and are simply objects with an associated callback.

## Getting Started

So when can you use the async local storage API, and what benefits does it provide?

Generally, if you are setting a property on the `request` or `response` handlers of a HTTP request, you can use async local storage. Also, if you want to persist some information during the lifetime of a request, this API comes in handy.

Let's now write some code to illustrate this:

```javascript
const express = require('express');
const axios = require('axios');
const { AsyncLocalStorage } = require('async_hooks');
const uuid = require('uuid/v4');

const asyncLocalStorage = new AsyncLocalStorage();

const requestMiddleware = async (req, res, next) => {
    await asyncLocalStorage.run(new Map(), () => {
        asyncLocalStorage.getStore().set('reqId', uuid());
        next();
      });
}
const app = express();
app.use(express.json());
app.use(requestMiddleware);

app.post('/test', (req, res) => {
    const store = asyncLocalStorage.getStore();
    const reqId = store.get('reqId');
    console.log('reqId<<<', reqId);
    res.status(200).json({
        message: 'Testing response worked 😄',
        requestId: reqId
    })
});


app.listen('4000', () => {
    console.log('listening on port 4000...')
});
```

This code sets up a trivial Node.js API with a `test` endpoint that returns the captured request id of the current request.

To capture the request, we create an instance of the async local storage API as follows:

```javascript
const asyncLocalStorage = new AsyncLocalStorage();
```

We then create our request middleware that will capture our HTTP request. Inside it, we use the `asyncLocalStorage.run()` method to set up our async store.

The first argument of the method is a store, which in our case is a javascript map, while the second is a callback and the last (not passed in our case), an optional list of arguments that are passed to the callback. The store is only accessible inside the callback or in async operations created by the callback.

The `asyncLocalStorage.getStore()` method returns the contents of the store object which in our case is a map. In our example, we generate a new request Id and set it in the map.

To use our middleware, we register it using the express `app.use()` syntax as follows. This makes the async local storage store available down our request pipeline.

```javascript
app.use(requestMiddleware);
```

Since we have access to the store, we can simply fetch the request id from the store in the `/test` endpoint and log it as follows:

```javascript
const store = asyncLocalStorage.getStore();
const reqId = store.get('reqId');
```

When you run this code, and make a request on a REST client you should get a response similar to this:

![Screenshot of REST Client](/images/getting-started-with-async-local-storage/screenshot01.png)

You should also get the request id logged on the terminal.

## Conclusion

What I have shared, is a contrived example showing what is possible with the async local storage API. There is certainly a wider field for its application.

I conjecture that the order of problems that are currently solved by cls-hooked could be well solved by this API. <a rel="nofollow noopener" target="_blank" href="https://nodejs.org/api/async_hooks.html#async_hooks_class_asynclocalstorage">The official documentation </a> is probably the best place to start.

Happy coding 😄
