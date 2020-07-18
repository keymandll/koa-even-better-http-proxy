# koa-even-better-http-proxy

Koa middleware to proxy request to another host and pass response back. Based on [koa-better-http-proxy](https://github.com/nsimmons/koa-better-http-proxy).

## Context

First of all, thanks a lot to the original author(s). It's a pretty cool library.

### Why?!

...we have to have yet another version of this proxy, you may ask. The answer is:

 1. The `koa-better-http-proxy` has been last updated 3 years ago.
 2. There was an issue reported about more than a year ago (2019), which also affected me: https://github.com/nsimmons/koa-better-http-proxy/issues/22
 3. The above issue with many other reported were not actioned so it was safe to assume the library was no longer maintained.

As I was using the library, I decided to fork and fix #2. Why I have not even attempted to submit a pull request?

 1. I simply cannot wait for someone else (not even a day). I suspect based on someone else's seemingly sensible pull request that has not been actioned since 2 years now, I could wait for ever. I cannot afford that.
 2. I did not want to write test(s) for this simple change (I'm lazy) - I tested it manually though... LOL

### What to expect?

Have no expectations at all. I'm only going to implement changes, bug fixes, etc. that affect **my project**. If it happens to be a good fit for you, great. Otherwise, it is better you look for an alternative library.

### So what has changed (so far)?

If the `Content-Type` or a request is `multipart/form-data` the value of the `parseReqBody` option is forced to be `false` so multipart messages are proxied properly. In any other scenario, `parseReqBody` works just as before. 

## Install

```bash
$ npm install koa-even-better-http-proxy --save
```

## Usage
```js
proxy(host, options);
```

To proxy URLS to the host 'www.google.com':

```js
var proxy = require('koa-even-better-http-proxy');
var Koa = require('koa');

var app = new Koa();
app.use(proxy('www.google.com'));
```

If you wish to proxy only specific paths, you can use a router middleware to accomplish this. See [Koa routing middlewares](https://github.com/koajs/koa/wiki#routing-and-mounting).

### Options

#### port

The port to use for the proxied host.

```js
app.use(proxy('www.google.com', {
  port: 443
}));
```

#### headers

Additional headers to send to the proxied host.

```js
app.use(proxy('www.google.com', {
  headers: {
    'X-Special-Header': 'true'
  }
}));
```

#### preserveReqSession

Pass the session along to the proxied request

```js
app.use(proxy('www.google.com', {
  preserveReqSession: true
}));
```

#### proxyReqPathResolver (supports Promises)

Provide a proxyReqPathResolver function if you'd like to
operate on the path before issuing the proxy request.  Use a Promise for async
operations.

```js
app.use(proxy('localhost:12345', {
  proxyReqPathResolver: function(ctx) {
    return require('url').parse(ctx.url).path;
  }
}));
```

Promise form

```js
app.use(proxy('localhost:12345', {
  proxyReqPathResolver: function(ctx) {
    return new Promise(function (resolve, reject) {
      setTimeout(function () {   // do asyncness
        resolve(fancyResults);
      }, 200);
    });
  }
}));
```

#### filter

The ```filter``` option can be used to limit what requests are proxied.  Return ```true``` to execute proxy.

For example, if you only want to proxy get request:

```js
app.use(proxy('www.google.com', {
  filter: function(ctx) {
     return ctx.method === 'GET';
  }
}));
```

#### userResDecorator (supports Promise)

You can modify the proxy's response before sending it to the client.

##### exploiting references
The intent is that this be used to modify the proxy response data only.

Note:
The other arguments (proxyRes, ctx) are passed by reference, so
you *can* currently exploit this to modify either response's headers, for
instance, but this is not a reliable interface. I expect to close this
exploit in a future release, while providing an additional hook for mutating
the userRes before sending.

##### gzip responses

If your proxy response is gzipped, this program will automatically unzip
it before passing to your function, then zip it back up before piping it to the
user response.  There is currently no way to short-circuit this behavior.

```js
app.use(proxy('www.google.com', {
  userResDecorator: function(proxyRes, proxyResData, ctx) {
    data = JSON.parse(proxyResData.toString('utf8'));
    data.newProperty = 'exciting data';
    return JSON.stringify(data);
  }
}));
```

```js
app.use(proxy('httpbin.org', {
  userResDecorator: function(proxyRes, proxyResData) {
    return new Promise(function(resolve) {
      proxyResData.funkyMessage = 'oi io oo ii';
      setTimeout(function() {
        resolve(proxyResData);
      }, 200);
    });
  }
}));
```

#### limit

This sets the body size limit (default: `1mb`). If the body size is larger than the specified (or default) limit,
a `413 Request Entity Too Large`  error will be returned. See [bytes.js](https://www.npmjs.com/package/bytes) for
a list of supported formats.

```js
app.use(proxy('www.google.com', {
  limit: '5mb'
}));
```

#### proxyReqOptDecorator  (supports Promise form)

You can mutate the request options before sending the proxyRequest.
proxyReqOpt represents the options argument passed to the (http|https).request
module.

```js
app.use(proxy('www.google.com', {
  proxyReqOptDecorator: function(proxyReqOpts, ctx) {
    // you can update headers
    proxyReqOpts.headers['content-type'] = 'text/html';
    // you can change the method
    proxyReqOpts.method = 'GET';
    // you could change the path
    proxyReqOpts.path = 'http://dev/null'
    return proxyReqOpts;
  }
}));
```

You can use a Promise for async style.

```js
app.use(proxy('www.google.com', {
  proxyReqOptDecorator: function(proxyReqOpts, ctx) {
    return new Promise(function(resolve, reject) {
      proxyReqOpts.headers['content-type'] = 'text/html';
      resolve(proxyReqOpts);
    })
  }
}));
```

#### proxyReqBodyDecorator  (supports Promise form)

You can mutate the body content before sending the proxyRequest.

```js
app.use(proxy('www.google.com', {
  proxyReqBodyDecorator: function(bodyContent, ctx) {
    return bodyContent.split('').reverse().join('');
  }
}));
```

You can use a Promise for async style.

```js
app.use(proxy('www.google.com', {
  proxyReqBodyDecorator: function(proxyReq, ctx) {
    return new Promise(function(resolve, reject) {
      http.get('http://dev/null', function (err, res) {
        if (err) { reject(err); }
        resolve(res);
      });
    })
  }
}));
```

#### https

Normally, your proxy request will be made on the same protocol as the original
request.  If you'd like to force the proxy request to be https, use this
option.

```js
app.use(proxy('www.google.com', {
  https: true
}));
```

#### preserveHostHdr

You can copy the host HTTP header to the proxied express server using the `preserveHostHdr` option.

```js
app.use(proxy('www.google.com', {
  preserveHostHdr: true
}));
```

#### parseReqBody

The ```parseReqBody``` option allows you to control parsing the request body.
For example, disabling body parsing is useful for large uploads where it would be inefficient
to hold the data in memory.

This defaults to true in order to preserve legacy behavior.

When false, no action will be taken on the body and accordingly ```req.body``` will no longer be set.

Note that setting this to false overrides ```reqAsBuffer``` and ```reqBodyEncoding``` below.

```js
app.use(proxy('www.google.com', {
  parseReqBody: false
}));
```

#### reqAsBuffer

Note: this is an experimental feature.  ymmv

The ```reqAsBuffer``` option allows you to ensure the req body is encoded as a Node
```Buffer``` when sending a proxied request.   Any value for this is truthy.

This defaults to to false in order to preserve legacy behavior. Note that
the value of ```reqBodyEnconding``` is used as the encoding when coercing strings
(and stringified JSON) to Buffer.

Ignored if ```parseReqBody``` is set to false.

```js
app.use(proxy('www.google.com', {
  reqAsBuffer: true
}));
```

#### reqBodyEncoding

Encoding used to decode request body. Defaults to ```utf-8```.

Use ```null``` to preserve as Buffer when proxied request body is a Buffer. (e.g image upload)
Accept any values supported by [raw-body](https://www.npmjs.com/package/raw-body#readme).

The same encoding is used in the userResDecorator method.

Ignored if ```parseReqBody``` is set to false.

```js
app.use(proxy('httpbin.org', {
  reqBodyEncoding: null
}));
```


#### timeout

By default, node does not express a timeout on connections.
Use timeout option to impose a specific timeout.
Timed-out requests will respond with 504 status code and a X-Timeout-Reason header.

```js
app.use(proxy('httpbin.org', {
  timeout: 2000  // in milliseconds, two seconds
}));
```
