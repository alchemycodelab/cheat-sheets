Express
===

## Creating an App

An express app is created by calling the "required" express function:

```js
const app = require('express')();
```

### Add Middleware and Routes

Add middlewares and routers using `app.use` with optional mounting path:

```js
app.use(morgan('dev'));
app.use(cors());
app.use(express.static(publicDir));

const widgets = require('./lib/routes/widgets');
app.use('/api/widgets', widgets);
```

### Add App as Listener to HttpServer

While there is a convience `.listen` method on `app`, we usually export `app` so we
can manage testing and running the server differently. 

`app` is an http `requestListener` function and can be passed to `http.createServer`:

```js
const http = require('http');
const app = require('./lib/app');
const server = http.createServer(app);
```

### Starting the Server

Call `.listen(port)` on the httpServer, using `process.env.PORT` and optionally
providing a default for development:

```js
const port = process.env.PORT || 3000;

server.listen(port, () => {
    console.log('server running', server.address());
});
```

## Creating a Router

An Express router is created by calling the `express.Router` function:

```js
const router = require('express').Router();
```

Specific methods can then be added to the router, which will be exported
and added to the `app`:

```js
router
    .get('/', (req, res, next) => {

    })
    .post('/', bodyParser, (req, res, next) => {

    });

module.exports = router;
```

## Middleware

Common middleware to use from `npm`:

* `express.static` - server a folder's contents as files
* `body-parser` - add to parse JSON to `req.body`
* `morgan` - http route logging
* `cors` - CORS support for running app and server on different hosts
* `ensureAuth` - middleware you write to check that user has a token indicating they are authenticated

Middleware is typically "required" and then called (including any options) before adding to `app.use()`.
Either call immediate when requiring:

```js
const cors = require('cors')();
const bodyParser = require('body-parser').json();
const morgan = require('morgan')('dev');
const ensureAuth = require('./auth/ensureAuth')();

app.use(cors);
app.use(bodyParser);
app.use(morgan);
app.use('/api/widgets', ensureAuth, widgets);
```

or call when passing to `app.use`:

```js
const cors = require('cors');
const bodyParser = require('body-parser');
const morgan = require('morgan');
const ensureAuth = require('./auth/ensureAuth');

app.use(cors());
app.use(bodyParser.json());
app.use(morgan('dev'));
app.use('/api/widgets', ensureAuth(), widgets);
```

Try and resuse middleware instances within a module when it makes sense:

```js
const ensureAuth = require('./auth/ensureAuth')();

app.use('/api/widgets', ensureAuth, widgets);
app.use('/api/products', ensureAuth, products);
app.use('/api/stores', ensureAuth, stores);
```

Use a more selective approach if possible when applying middleware. 
For example, add `bodyParser` to `post` routes instead of all routes:

```js
.post('/', bodyParser, (req, res, next) => {/*...*/});
```

## Url Paths

When using the http methods on `app` or `router`, the path is an exact match except
that it will also match if there is a trailing slash:

```js
// matches "/api/weasels" and "/api/weasels/":

app.get('/api/weasels', ... );

// does not match: "/api/weasels/foo"
```

When using a "mounting path" on `app.use`, the path needs to match the 
_start of the requested path_:

```js
// matches "/api/weasels", "/api/weasels/", "/api/weasels/foo", etc.
// basically any request starting with "/api/weasels"

app.use('/api/weasels', weasels);
```

When adding a router to a use with a mount path, the routes inside the
router are _supplemental_ to the mounting path:

```js
// weasels router from previous example

// matches "/api/weasels/ab43b2b2"
router.get('/:id', ...);
```

Use a single slash to match the mounting path directly:

```js
// weasels router from previous example

// matches "/api/weasels" or "/api/weasels/"
router.get('/', ...);
```

## Route and Query Parameters

### Parameters

Parameters are dynamic path placeholders that will be automatically added 
to the `req.params` property.

```js
.get('/:id', (req, res, next) => {
    const id = req.params.id;
    // ...
});
```

### Query parameters

Url query parameters represent refinements to the resource being requested.

Express will be automatically parse and add them to the `req.query` property:

```js
// /api/widgets?type=sprocket&metal=aluminum

.get('/', (req, res, next) => {
    const type = req.query.type; //"sprocket"
    const metal = req.query.metal; //"aluminum"
    // ...
})
```

## Errors

### Error Handler

Data API servers should return JSON error messages, not text or html. To create a custom
error handler, create a middleware that uses the four parameter function definition: 
`error, request, response, next`:

```js
function errorHandler(err, req, res, next) {

    // Default code and error
    let code = 500, error = 'Internal Server Error';

    // Catch Mongoose Validation Errors
    if(err.name === 'ValidationError' || err.name === 'CastError') {
        code = 400;
        error = err.errors.name.message;
        console.log(code, error);
    }
    // A specific error raised by our code
    else if(err.code) {
        // by convention, we're passing in an object
        // with a code property === http statusCode
        // and a error property === message to display
        code = err.code;
        error = err.error;
        console.log(code, error);
    }
    // An unexpected error
    // (stays code 500 'Internal Server Error')
    else {
        // but log out real error on server
        console.log(err);
    }

    res.status(code).send({ error });
};
```

Delineate between 400 level codes that deal with requestor issues, 
and 500 level codes that deal with server errors. Don't forward actual `500`
error message as it represents an unexpected error and it may
reveal inner workings of our server that compromise security.

### Routing to the Error Handler

If your routes could have failures (usually always), you need to include `next` 
and use it to route unexpected (or Mongoose) errors to the error handler:

```js
.get('/', (req, res, next) => {
    MyModel.find()
        .then(models => res.send(models))
        .catch(next);
});
```

`400` level errors are rooted in something wrong with the request and your code
will identify these errors and send the approriate code and message 
to the error handler:

```js
.get('/:id', (req, res, next) => {
    MyModel.findById(req.params.id)
        .then(model => {
            if(!model) next({ code: 404, message: `myModel ${req.params.id} not found`)
            else res.send(model);
        })
        .catch(next);
});
```
