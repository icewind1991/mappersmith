[![npm version](https://badge.fury.io/js/mappersmith.svg)](http://badge.fury.io/js/mappersmith)
[![Build Status](https://travis-ci.org/tulios/mappersmith.svg?branch=master)](https://travis-ci.org/tulios/mappersmith)
# Mappersmith

**Mappersmith** is a lightweight, isomorphic, dependency-free, rest client mapper for javascript. It helps you map your API to use at the client and/or server, giving you all the flexibility you want to customize requests or write your own gateways.

https://www.npmjs.com/package/mappersmith

## Browser support

Mappersmith was designed considering modern browsers. However, all the methods used can be included by a shim such as [es5-shim](https://github.com/es-shims/es5-shim).

## Install

#### NPM

```sh
npm install mappersmith
```

#### Browser

Download the tag/latest version from the build folder.

#### Build from the source

Install the dependencies

```sh
npm install
```

Build

```sh
npm run build
npm run release # for minified version
```

## Requiring in Node.js

```javascript
var Mappersmith = require('mappersmith/node');
```

## Usage

To create a client for your API, you will need to provide a simple manifest, which must have `host` and `resources` keys. Each resource has a name and a list of methods with its definitions, like:

```javascript
var manifest = {
  host: 'http://my.api.com',
  resources: {
    Book: {
      all:  {path: '/v1/books.json'},
      byId: {path: '/v1/books/{id}.json'}
    },
    Photo: {
      byCategory: {path: '/v1/photos/{category}/all.json'}
    }
  }
}
```

You can specify an HTTP method for every API call, but if you don't, `GET` will be used. For instance, let's say you can save a photo:

```javascript
...
Photo: {
  save: {method: 'POST', path: '/v1/photos/{category}/save'}
}
...
```

With the manifest in your hands, you are able to _forge_ your client:

```javascript
var Client = Mappersmith.forge(manifest)
```

And then, use it as defined:

```javascript
// without callbacks
Client.Book.byId({id: 3})

// with all callbacks
Client.Book.byId({id: 3}, function(data, stats) {
  // success callback
}).fail(function(request, err) {
  // fail callback
}).complete(function() {
  // complete callback, it will always be called
})
```

__Mappersmith supports Promises, check [how to enable](#using-with-promises) in a section bellow__

#### Success callback arguments

The success callback will receive two arguments: the _first one_ will be `data`, returned by your API; and the _second one_ will be a `stats` object. The stats object hold information of the request, like the elapsed time between your call and callback execution.

Be aware that plugins hooked to Mappersmith can include more stats into this object, like [CachedGateway](https://github.com/tulios/mappersmith-cached-gateway) which includes if the call got an cache _hit_ or _miss_.

The __default stats__ in the object are: `url`, `params`, `timeElapsed` and `timeElapsedHumanized`. Example:

```javascript
{
  url: 'http://my.api.com/v1/books.json?language=en',
  host: 'http://my.api.com',
  path: '/v1/books.json?language=en',
  params: {language: 'en'},
  timeElapsed: 6.745000369846821,
  timeElapsedHumanized: '6.75 ms'
}
```

#### Fail callback arguments

The fail callback will receive in the first argument the requested resource, which is an object that contains the requested URL, host, path and params. From the second argument and beyond it will receive the error objects from the specific gateway implementations.

```javascript
...
fail(function(request, err) {
  console.log(request.url) // 'http://my.api.com/v1/books/3.json'
  console.log(request.params) // {id: 3}
});
```

#### Parameters

If your method doesn't require any parameter, you can just call it without them:

```javascript
Client.Book.all() // http://my.api.com/v1/books.json
```

Every parameter that doesn't match a pattern (`{parameter-name}`) in `path` will be sent as part of the query string:

```javascript
Client.Book.all({language: 'en'}) // http://my.api.com/v1/books.json?language=en
```

#### Default parameters

It is possible to configure default parameters for your resources, just use the key `params` in the definition. It will replace params in the URL or include query strings, for example, imagine that our manifest has the method __byYear__ in the resource __Photo__:

```javascript
...
Photo: {
  byYear: {
    path: '/v1/photos/{year}.json',
    params: {year: new Date().getFullYear(), category: 'cats'}
  }
}
...
```

If we call it without any params and `new Date().getFullYear()` is 2015, it will generate the following URL:

```javascript
Client.Photo.byYear();
// http://my.api.com/v1/photos/2015.json?category=cats
```

And, of course, we can override the defaults:

```javascript
Client.Photo.byYear({category: 'dogs'});
// http://my.api.com/v1/photos/2015.json?category=dogs
```

#### Message Body

To send values in the request body (usually for POST or PUT methods) you will use the special parameter `body`:

```javascript
Client.Photo.save({
  category: 'family',
  body: {year: 2015, tags: ['party', 'family']}
})
```

It will create a _urlencoded_ version of the object (`year=2015&tags[]=party&tags[]=family`). If the `body` used
is not an object it will use the original value. If `body` is not possible as a special parameter
for your API you can configure it with another value, just pass the new name as the third argument
of method forge:

```javascript
var Client = Mappersmith.forge(manifest, Mappersmith.VanillaGateway, 'data')
...
Client.Photo.save({
  category: 'family',
  data: {year: 2015, tags: ['party', 'family']}
})
```

#### Processors

You can specify functions to process returned data before they are passed to success callback:

```javascript
...
Book: {
  all:  {
    path: '/v1/books.json',
    processor: function(data) {
      return data.result;
    }
  }
}
...
```

#### Alternative host

There are some cases where a resource method reside in another host, in those cases you can use the `host` key to configure a new host or to disable the resolution.

```javascript
var manifest = {
  host: 'http://new-host.com/api/v2',
  resources: {
    MyResouce: {
      all: {path: '/all.json'},
      byId: {path: '/{id}.json', host: 'http://old-host.com/api/v1'},
      other: {path: '{url}', host: false}
    }
  }
}

var Client = Mappersmith.forge(manifest);
Client.MyResource.all()
// http://new-host.com/api/v2/all.json

Client.MyResource.byId({id: 1})
// http://old-host.com/api/v1/1.json

Client.MyResource.other({url: 'http://host.com/other/'})
// http://host.com/other/
```

It's also possible to disable the host resolution for all resources, using the current URL, just assign `false` to the host key and remember to start your resources paths with `/`.

```javascript
var manifest = {
  host: false,
  resources: {
    MyResouce: {
      all: {path: '/all.json'}
    }
  }
}

var Client = Mappersmith.forge(manifest);
Client.MyResource.all()
// /all.json
```

#### <a name="using-with-promises"></a> Using with Promises

To disable the callback API and enable Promises you must turn on the flag `USE_PROMISES`.

```javascript
Mappersmith.Env.USE_PROMISES = true;
```

After that, you can forge your client and assume that every method will return a promise.

```javascript
var Client = Mappersmith.forge(manifest);

Client.Book.byId({id: 3}).then(function(response) {
  console.log(response.data); // data returned by your API
  console.log(response.stats); // stats object with request information

}).catch(function(err) {
  console.log(err.response); // requested resource, same as stats
  console.log(err.err); // array of errors given by gateway
});

// other example

Client.Book.all().then(function(response) {
  return response.data;

}).then(function(data) {
  console.log(data);
})
```

The first callback, if provided, will be used as a "then" statement, example:

```javascript
Client.Book.all(function() {
  console.log(1);

}).then(function() {
  console.log(2);
});

// output:
// 1
// 2
```

It is important to note that Mappersmith __does not apply__ any polyfills. If you are using this with a browser that doesn't support Promises, please apply the polyfill first. One option can be [then/promises](https://github.com/then/promise)

#### Compact Syntax
If you find tiring having to map your API methods with hashes, you can use our incredible compact syntax:

```javascript
...
Book: {
  all: 'get:/v1/books.json', // The same as {method: 'GET', path: '/v1/books.json'}
  byId: '/v1/books/{id}.json' // The default is GET, as always
},
Photo: {
  // The same as {method: 'POST', path: '/v1/photos/{category}/save.json'}
  save: 'post:/v1/photos/{category}/save'
}
...
```

__A downside is that you can't use processor functions with compact syntax.__

## Gateways

**Mappersmith** allows you to customize the transport layer. There are gateways for browser and server (Nodejs). You can use the default `Mappersmith.VanillaGateway` (client only), the included `Mappersmith.JQueryGateway` (client only), `NodeVanillaGateway` (server only) or write your own version. Check the list of [available gateways](#gateway-implementations) at the bottom of the readme.

#### How to write one?

```javascript
var MyGateway = Mappersmith.createGateway({
  get: function() {
    // you will have:
    // - this.url
    // - this.host
    // - this.path
    // - this.params
    // - this.body
    // - this.opts
  },

  post: function() {
  }

  // and other HTTP methods
})
```

#### How to change the default?

Just provide an object created with `Mappersmith.createGateway` as the second argument of the method `forge`:

```javascript
var Client = Mappersmith.forge(manifest, Mappersmith.JQueryGateway)
```

#### Specifics of each gateway

You can pass options for the gateway implementation that you are using. For example, if we are using the `Mappersmith.JQueryGateway` and want one of our methods to use `jsonp`, we can call it like:

```javascript
Client.Book.byId({id: 2}, function(data) {}, {jsonp: true})
```

The third argument is passed to the gateway as `this.opts` and, of course, the accepted options vary by each implementation. The default gateway, `Mappersmith.VanillaGateway`, accepts a `configure` callback:

```javascript
Client.Book.byId({id: 2}, function(data) {}, {
  configure: function(request) {
    // do whatever you want
  }
})
```

#### Global configurations and URL matching

Imagine that you are using `Mappersmith.JQueryGateway` and all of your methods must be called with `jsonp` or use a special header, it will be incredibly boring add those configurations every time. Global configurations allow you to configure gateway options and a processor that will be used for every method. Keep in mind that the processor configured in the resource will be prioritized instead to global, for example:

```javascript
var manifest = {
  host: 'http://my.api.com',
  rules: [
    { // This is our global configuration
      values: {
        gateway: {jsonp: true},
        processor: function(data) { return data.result }
      }
    }
  ],
  resources: {
    Book: {
      all:  {path: '/v1/books.json'},
      byId: {path: '/v1/books/{id}.json'}
    },
    Photo: {
      byCategory: {path: '/v1/photos/{category}/all.json'}
    }
  }
}
```

It is possible to add some configurations based on matches in the URLs, let's include a header for every book URL:

```javascript
...
rules: [
  { // This is our global configuration
    values: {
      gateway: {jsonp: true},
      processor: function(data) { return data.result }
    }
  },
  { // This will only be applied when the URL matches the regexp
    match: /\/v1\/books/,
    values: {
      gateway: {headers: {'X-MY-HEADER': 'value'}}
    }
  }
]
...
```

Just keep in mind that the configurations and processors will be prioritized by their order, and the global configurations does not have a `match` key.

## <a name="gateway-implementations"></a> Gateway Implementations

The gateways listed here are available through the `Mappersmith` namespace.

### VanillaGateway

__Client Only__. The default gateway - it uses plain `XMLHttpRequest`. Accepts a `configure` callback that allows you to change the request object before it is used.

Available methods:

- :ok: GET
- :ok: POST
- :ok: PUT
- :ok: DELETE
- :ok: PATCH

Available options:

- emulateHTTP: sends request as POST with `_method` in the body and `X-HTTP-Method-Override` header, both with requested method as value. (default `false`)


### JQueryGateway

__Client Only__. It uses `$.ajax` and accepts an object that will be merged with `defaults`. It doesn't include **jquery**, so you will need to include that in your page.

Available methods:

- :ok: GET
- :ok: POST
- :ok: PUT
- :ok: DELETE
- :ok: PATCH

Available options:

- emulateHTTP: sends request as POST with `_method` in the body and `X-HTTP-Method-Override` header, both with request method as value. (default `false`)

### NodeVanillaGateway

__Server Only__. It uses the module `http` and accepts an object that will be merged with `defaults`.

How to access this gateway?

```javascript
var Mappersmith = require('mappersmith/node');
Mappersmith.node.NodeVanillaGateway;
```

Available methods:

- :ok: GET
- :ok: POST
- :ok: PUT
- :ok: DELETE
- :ok: PATCH

Available options:

- emulateHTTP: sends request as POST with `_method` in the body and `X-HTTP-Method-Override` header, both with request method as value. (default `false`)

## Extras

For gateways with transparent cache functionalities and different cache stores, take a look at:

[https://github.com/tulios/mappersmith-cached-gateway](https://github.com/tulios/mappersmith-cached-gateway)

for a layer on top of your objects/responses to help with common annoyances which the javascript world provides daily, take a look at:

[https://github.com/tulios/mappersmith-object](https://github.com/tulios/mappersmith-object)

## Tests

### Client

`npm run test-browser` or `SINGLE_RUN=true npm run test-browser`

### Server

`npm run test-node`

### To run both tests

`npm test`

## Compile and release

* Compile: `npm run build`
* Release (minified version): `npm run release`

## Contributors

Check it out!

https://github.com/tulios/mappersmith/graphs/contributors

## License

See [LICENSE](https://github.com/tulios/mappersmith/blob/master/LICENSE) for more details.
