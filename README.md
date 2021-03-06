# AWS Lambda Router for NodeJS

[![Build Status](https://travis-ci.org/doomhz/node-lambda-req.svg?branch=master)](https://travis-ci.org/doomhz/node-lambda-req) [![Coverage Status](https://coveralls.io/repos/github/doomhz/node-lambda-req/badge.svg?branch=master)](https://coveralls.io/github/doomhz/node-lambda-req?branch=master)


A collection of tools to handle **ApiGateway requests** and **direct function invocation calls** on AWS Lambda.
Use it as a stand-alone lib in a Lambda function or plugged-in to **Apex** or any other serverless framework on NodeJS.
I'd recommend deploying and managing your Lambda functions with **Apex and LambdaReq** since they play nicely together.


## Installation

`npm install lambda-req --save`

## Deploy it with Apex

```javascript
import LambdaReq from 'lambda-req'

const lambda = new LambdaReq()

// set APIGateway handlers
lambda.get('/lreqex_hello', (req, router)=> {
  const { params } = req
  return { message: 'hello world!', params }
})
lambda.post('/lreqex_hello', (req, router)=> {
  const { params } = req
  return { message: 'hello world!', params }
})

// set direct invocation handlers
lambda.proxy('hello', (req, router)=> {
  const { params } = req
  return { message: 'hello world!', params }
})

export default lambda.invoke
```

## Use it stand-alone on Lambda

```javascript
import LambdaReq from 'lambda-req'

// initialize Lambda with no params, pass them later from handler
const lambda = new LambdaReq()

// set handlers
lambda.get('/v1/test', (req, router)=> {})
lambda.proxy('migrate', (req, router)=> Promise.resolve({}))

// export the handler
// pass the event params on invocation time
export { handler: lambda.invoke }
```

## Invoke other Lambdas

```javascript
import LambdaReq, { LambdaProxy, LambdaReqError } from 'lambda-req'

const lambda = new LambdaReq()

// set APIGateway handlers
lambda.get('/lreqex_proxy', (req, router)=> {
  const { params } = req
  
  const proxy = new LambdaProxy()
  
  // Invoke another Lambda in the same AWS VPC
  return proxy.invoke('lreqex_hello', 'hello', params)
  .then((response)=> {
    return { message: 'Proxy response from lreqex_hello', response }
  })
  .catch((err)=> {
    console.error(err)
    throw new LambdaReqError({
      message: {
        error: {
          code: 'lambdaInvocationError',
          message: 'lreqex_hello Lambda is unresponsive.'
        }
      }
    })
  })
})

export default lambda.invoke
```

See [more examples](examples/).


## API


## LambdaReq

This is the Lambda router object. It binds routes and proxies events for a Lambda handler.


### .constructor([event = `Object`], [context = `Object`], [callback = `function`])

Pass in the [lambda handler](http://docs.aws.amazon.com/lambda/latest/dg/nodejs-prog-model-handler.html) `event`, `context` and `callback` on object initialization. However, this is optional since they can be sent diectly on hanlder invocation.

Node 4.3:

```javascript
const LambdaReq = require('lambda-req').default

function handler (event, context, callback) {
  const lambda = new LambdaReq(event, context, callback)
  lambda.invoke()
}

module.exports = { handler }
```

With Babel:

```javascript
import LambdaReq from 'lambda-req'

function handler (event, context, callback) {
  const lambda = new LambdaReq(event, context, callback)
  lambda.invoke()
}

export { handler }
```


### .get|.post|.put|.delete|.options(path = `String`, handler = `function`)

Shorthand methods for binding **APIGateway route** handlers. **ALL APIGateway params** (path, query, body) are passed through the `params` key on the `req` argument. A router instance is passed as the second handler arg. `router._event` and `router._context` are mapped to the original handler params.

```javascript
const lambda = new LambdaReq(event, context, callback)

lambda.get('/test', handler)
lambda.post('/test', handler)
lambda.put('/test', handler)
lambda.delete('/test', handler)

lambda.invoke()

async function handler (req, router) {
  const { params, headers } = req
  return {}
}
```

A request to `https://api-gateway-id.execute-api.eu-west-1.amazonaws.com/prod/test` will fire up the `GET` handler on the `/test` route.


### .proxy(name = `String`, handler = `function`)

Shorthand method for binding **Lambda direct invocation** handlers. Used for maintenance tasks (i.e. db migrations, cronjobs) and internal Lambda calls. Each proxy handler has a unique name, passed as a `command` key on the Lambda event. A direct call, proxies all the event data to the handler.

```javascript
const lambda = new LambdaReq(event, context, callback)

lambda.proxy('db_migrate', handler)

lambda.invoke()

async function handler (req, router) {
  const { params } = req
  return {}
}
```

Invoke this functions from command line with [aws-cli](http://docs.aws.amazon.com/cli/latest/reference/lambda/index.html):

```bash
#!/usr/bin/env bash

PAYLOAD='{"command":"db_migrate","params":{"name":"users"}}'

aws lambda invoke \
  --function-name myFunction \
  --payload $PAYLOAD \
  out.log
```

or through [aws-sdk](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Lambda.html):

```javascript
import AWS from 'aws-sdk'

const config = {
  FunctionName: 'myFunction',
  Payload: JSON.stringify({ command: 'db_migrate', params: { name: 'users' } })
}
const lambda = new AWS.Lambda()
lambda.invoke(config).promise().then(response => JSON.parse(response.Payload))
```


### .invoke([event = `Object`], [context = `Object`], [callback = `function`])

Invokes the handlers that match the current Lambda route from APIGateway or by a direct invocation call. Optionally, Lambda handler params will be passed along. It's recommended to export this method as a Lambda handler.

```javascript
import LambdaReq from 'lambda-req'

const lambda = new LambdaReq()

lambda.get('/test', ()=> {})
lambda.proxy('db_migrate', ()=> {})

export { handler: lambda.invoke }
```

or with `Apex` and `WebPack`

```javascript
import LambdaReq from 'lambda-req'

const lambda = new LambdaReq()

lambda.get('/test', ()=> {})
lambda.proxy('db_migrate', ()=> {})

export default lambda.invoke
```


### .isApiGateway `Bool`

It returns `true` if the request was an APIGateway invocation. Internally, checks if an HTTP method is set on the Lambda event.

```javascript
const lambda = new LambdaReq(event, context, callback)

if (lambda.isApiGateway) {
  // it's a HTTP call
}
```


### .isProxy `Bool`

It return `true` if the request was a direct Lambda invocation. Internally, checks if a property `command` is set on the Lambda event.

```javascript
const lambda = new LambdaReq(event, context, callback)

if (lambda.isProxy) {
  // it's a direct invocation
}
```


### .params `Object`

Has all the params coming from `event.queryStringParameters`, `event.pathParameters` and `event.body` for an APIGateway call.
Stores all the params coming from `event.params` for a direct invocation.
Referenced through `req.params` on a handler.


```javascript
const lambda = new LambdaReq(event, context, callback)

lambda.get('/test', (req)=> {
  const { params } = req
  // process APIGateway data
})
lambda.proxy('db_migrate', (req)=> {
  const { params } = req
  // process direct invocation data
})

if (lambda.params) {
  // process data
}
```


### .headers `Object`

Stores the HTTP headers from an APIGateway call.

```javascript
const lambda = new LambdaReq(event, context, callback)

lambda.get('/test', (req)=> {
  const { headers } = req
  if (headers['x-auth']) {
    // process user authentication
  }
})
```


### .currentRoute `String`

Returns the current invocation route.



## LambdaProxy

A wrapper for direct Lambda invocation calls. Sets the `command` and `parameters` on internal calls.

### .constructor([client = `Object` = `AWS.Lambda`])

### .invoke(functionName = `String`, command = `String`, [params = `Object`])

Call a Lambda internally, pass in the command and params.

```javascript
import { LambdaProxy } from 'lambda-req'

const proxy = new LambdaProxy()
proxy.invoke('myFunction', 'db_migrate', { name: 'users' })
.then((response)=> {
  return { message: 'Proxy response from myFunction', response }
})
.catch((err)=> {
  console.error('myFunction Lambda is unresponsive', err)
})
```



## LambdaReqError

A tiny wrapper for Lambda request errors.


### .constructor({ message, status })

The error has a message and a status that will be passed along to the APIGateway.

```javascript
lambda.get('/users', (req, router)=> {
  const { params } = req
  if (!params.id) {
    throw new LambdaReqError({
      message: {
        error: {
          code: 'userNotFound',
          message: 'User not found.'
        }
      },
      status: 404
    })
  } else {
    const user = await dbConnection.findUser(params.id)
    return { user }
  }
})
```


See [more examples](examples/).
