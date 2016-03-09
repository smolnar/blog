---
layout: post
title: Concurrency and Multi-threading with JavaScript
---

As of JavaScript's beginning, developers glancing over JavaScript code might get confused and think that JavaScript is a concurrent language. But you can't blame them. Just looking over this code, it sure looks concurrent.

```javascript
console.log('Started.');

var google = new XMLHttpRequest();
var github = new XMLHttpRequest();

google.onreadystatechange = function() {
  console.log('Google loaded.');
}

github.onreadystatechange = function() {
  console.log('Github loaded.');
}

google.open('GET', 'https://google.com', true); // true as async
github.open('GET', 'https://github.com', true);

google.send()
github.send();

console.log('Finished.')
```

Once you run this code in your browser, you'll probably end up with result similar to this one. *Those GitHub and Google messages may be swapped.*

```
Started.
Finished.
Github loaded
Google loaded.
```

Looks like that those callbacks run concurrently, but they actually aren't. JavaScript is faking this notion of concurrency with non-blocking IO operations, e.g. HTTP request, database calls, disk reads/writes and others and it's concurrency model is built on top of model called [Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop).

## Event Loop and Concurrency

JavaScript uses Event Loop model to handle asynchronous IO operations. Event Loop was explained over and over again, so I'll keep this simple.

1. When making blocking IO, e.g an HTTP request, you provide a callback to run after it finishes.
2. Once it's finished, the callback is enqueued for execution and that's when Event Loop comes in.
3. Event Loop waits for main code to finish, so stack needs to be empty and there's no other code running at the moment.
4. When stack is empty, Event Loop runs enqueued callbacks one by one, synchronously.

That's event loop in a nutshell. But make sure to look into [this great explanation of Event Loop in JavaScript by Philip Roberts](https://www.youtube.com/watch?v=8aGhZQkoFbQ). Understanding Event Loop and callbacks in JavaScript can make you code way more efficient.

So when running zero-time `setTimeout` in `asyncLog`, what actually happens?

```javascript
var asyncLog = function(message) {
  return setTimeout(function() { console.log(message); }, 0);
}

for(i = 0; i < 5; i++) {
  asyncLog(i.toString());
}

console.log('Printing numbers asynchonously');
```

It waits zero seconds, so it almost immediately enqueues callback for execution. Our code is still running, so as first thing it print's out `Printing numbers asynchonously` and then Event Loop runs callbacks. As a result, all numbers from 1 to 5 are printed. In order.

No concurrency, only non-blocking IO. There's only one single thread running at the moment and callbacks are executed in order they were enqueued by their respective [Web API](https://developer.mozilla.org/en-US/docs/Web/API) calls.

So when you run CPU intensive task in JavaScript, even as callback run by Event Loop, it's *always* going to hang the browser. *Always*. But there's a solution to make it run in background.

## Web Workers

> A web worker is a JavaScript that runs in the background, independently of other scripts, without affecting the performance of the page. You can continue to do whatever you want: clicking, selecting things, etc., while the web worker runs in the background.
>
> &mdash; <cite>[W3Schools](http://www.w3schools.com/html/html5_webworkers.asp)</cite>

An awesome feature of HTML5 which basicly let's you have multiple workers handling computation concurrently. It sounds like a wonderland, but there's a catch.

```javascript
// worker.js

doHeavyComputation = function(number) {
  // Do some heavy computation
}

onmessage = function(event) {
  var number = event.data;
  var result = doHeavyComputation(number);

  postMessage(result);
}
```

```javascript
// app.js

var worker = new Worker('worker.js');

// or worker.addEventListener('message', function() { ... })
worker.onmessage = function(event) {
  var result = event.data;

  console.log('Result received from worker.');

  // Do something with the result.
}

worker.postMessage(5)
```

Concurrency is real and workers spawn actual OS threads. So let's just sum up main features.

1. A worker and main script communicated via messaging interface with `postMessage` and `onmessage`.
2. Every object passed via `postMessage` is copied, so no data is shared between worker and main script. [Thread-safety](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers#About_thread_safety) is guaranteed and very hard to break.
3. Worker cannot access DOM and has its own dedicated scope without access to window. Another way of achieving thread-safety since DOM is not thread-safe.
4. When workers fail, they fail silently. Catching errors works with `onerror` callback.

And that's the catch &ndash; thread-safety. Since workers need to run in separate scope, you can't access features like `localStorage`, DOM or many others others. So eventually, you can't leverage power of Web Workers to run intensive data import into `localStorage` to improve your app performance.

So that's most of the key points about Web Workers. For more thourough introduction, see [MDN on Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers).

Dispite limitations due to thread-safety, do you see any applications for this? Everytime you think of moving some complex logic to server to compute the result, you might think about having that on client and leverage it's capabilities. 

But the one thing that actually bugs me about Web Workers is the inability to pass in a function directly as parameter instead of passing it individual files. But with Blob URLs, it's doable.

## Inline Web Workers

Basically, the solution is to create a little hack and serialize function for Web Worker.

```javascript
var InlineWorker = function() { };

InlineWorker.create = function(callback) {
  var code = callback.toString();
  var blob = new Blob([`var f = ${code}; f.apply(this);`]);
  var blobURL = URL.createObjectURL(blob);

  return new Worker(blobURL);
}
```

And after that you can create a worker just by passing in the function.

```javascript
var worker = InlineWorker.create(function() {
  onmessage = function(event) {
    console.log(`Worker: ${event.data}`);

    postMessage('Done.');
  }
});

worker.onmessage = function(event) {
  console.log(`Main: ${event.data.toString()}`);
}

worker.postMessage('Hello, world.');

// => Worker: Hello, world.
// => Main: Done.
```

But this approach has a bunch of pitfalls.

Hiding this implementation behind a curtain of classes and factory methods without exposing its logic to developer can cause false notion that passing in a function also copies the context and preserves closures. Since function is evaluated inside worker scope, there's no data sharing or closure technique applied. Therefore, this approach needs to be applied transparently to developer.

## Wrapping up

Web Workers are an awesome feature, but the sad thing about them is their limitation. In order to achieve bullet-proof thread-safety, you can't access `localStorage`, which might be very hand in some cases. For example, an app could be pushing a ton of data into `localStorage` in background without slowing down the main script and the main script would be continuously showing the results. But it's still very easy handy tool for performing CPU intensive task client-side, e.g. computing results for rendering engine. *So game on!*
