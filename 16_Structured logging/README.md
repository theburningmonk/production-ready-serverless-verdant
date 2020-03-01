# Module 19: Structured logging

## Apply structured logging to the project

<details>
<summary><b>Using a simple logger</b></summary><p>

I built a suite of tools to help folks build production-ready serverless applications while I was at DAZN. It's now open source: [dazn-lambda-powertools](https://github.com/getndazn/dazn-lambda-powertools).

One of the tools available is a very simple logger that supports structured logging (amongst other things).

So, first, let's install the logger for our project.

1. At the project root, run the command `npm install --save @dazn/lambda-powertools-logger` to install the logger.

Now we need to change all the places where we're using `console.log`.

2. Open `functions/get-index.js`, and replace `console.log` with use of the logger. Replace the entire `get-index.js` with the following.

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')
const Log = require('@dazn/lambda-powertools-logger')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']
const ordersApiRoot = process.env.orders_api

const awsRegion = process.env.AWS_REGION
const cognitoUserPoolId = process.env.cognito_user_pool_id
const cognitoClientId = process.env.cognito_client_id

let html

function loadHtml () {
  if (!html) {
    Log.info('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    Log.info('loaded')
  }
  
  return html
}

const getRestaurants = async () => {
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname, 
    path: url.pathname
  }

  aws4.sign(opts)

  const httpReq = http
    .get(restaurantsApiRoot)
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])

  if (opts.headers['X-Amz-Security-Token']) {
    httpReq.set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  }

  return (await httpReq).body
}

module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const view = {
    awsRegion,
    cognitoUserPoolId,
    cognitoClientId,
    dayOfWeek,
    restaurants,
    searchUrl: `${restaurantsApiRoot}/search`,
    placeOrderUrl: `${ordersApiRoot}`
  }
  const html = Mustache.render(template, view)
  const response = {
    statusCode: 200,
    headers: {
      'content-type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

3. Modify `functions/place-order.js` and replace `console.log` with use of the logger.

First, require the logger at the top of the file.

```javascript
const Log = require('@dazn/lambda-powertools-logger')
```

Replace the 2 instances of `console.log` with `Log.debug`.

On ln11:

```javascript
Log.debug(`placing order ID [${orderId}] to [${restaurantName}]`)
```

On ln27:

```javascript
Log.debug(`published 'order_placed' event into Kinesis`)
```

4. Modify `functions/notify-restaurant.js` and replace `console.log` with use of the logger.

First, require the logger at the top of the file.

```javascript
const Log = require('@dazn/lambda-powertools-logger')
```

Replace the 2 instances of `console.log` with `Log.debug`.

On ln21:

```javascript
Log.debug(`notified restaurant [${order.restaurantName}] of order [${order.orderId}]`)
```

On ln32:

```javascript
Log.debug(`published 'restaurant_notified' event to Kinesis`)
```

5. Run the integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that the functions are now logging in JSON

```
  When we invoke the GET / endpoint
SSM params loaded
invoking via handler function get-index
{"level":"INFO","message":"loading index.html..."}
{"level":"INFO","message":"loaded"}
    ✓ Should return the index page with 8 restaurants (1988ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (226ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
{"level":"DEBUG","message":"notified restaurant [Fangtasia] of order [e0018c8c-e7fe-5930-a298-793dbf3df179]"}
{"level":"DEBUG","message":"published 'restaurant_notified' event to Kinesis"}
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  When we invoke the POST /orders endpoint
invoking via handler function place-order
{"level":"DEBUG","message":"placing order ID [e229e607-e2bf-5a54-b065-ca27be2b7bbb] to [Fangtasia]"}
{"level":"DEBUG","message":"published 'order_placed' event into Kinesis"}
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
    ✓ Should return an array of 4 restaurants (141ms)


  7 passing (3s)
```

</p></details>

<details>
<summary><b>Disable debug logging in production</b></summary><p>

This logger allows you to control the default log level via the `LOG_LEVEL` environment variable. Let's configure the `LOG_LEVEL` environment such that we'll be logging at `INFO` level in production, but logging at `DEBUG` level everywhere else.

1. Open `serverless.yml`. Add a `custom` section, this should be at the same level as `provider` and `plugins`.

```yml
custom:
  stage: ${opt:stage, self:provider.stage}
  logLevel:
    prod: INFO
    default: DEBUG
```

`custom.stage` uses the `${xxx, yyy}` syntax to provide fall backs. In this case, we're saying "if a `stage` variable is provided via the CLI, e.g. `sls deploy --stage staging`, then resolve to `staging`; otherwise, fallback to `provider.stage` in this file (hence the `self` reference"

2. Still in the `serverless.yml`, under `provider` section, add the following

```yml
environment:
  LOG_LEVEL: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

After this change, the `provider` section should look like this:

```yml
provider:
  name: aws
  runtime: nodejs12.x
  stage: dev
  environment:
    LOG_LEVEL: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

This applies the `LOG_LEVEL` environment variable (used to decide what level the logger should log at) to all the functions in the project (since it's specified under `provider`).

It references the `custom.logLevel` object (with the `self:` syntax), and also references the `custom.stage` value (remember, this can be overriden by CLI options). So when the deployment stage is `prod`, it resolves to `self:custom.logLevel.prod` and `LOG_LEVEL` would be set to `INFO`.

The second argument, `self:custom.logLevel.default` provides the fallback if the first path is not found. If the deployment stage is `dev`, it'll see that `self:custom.logLevel.dev` doesn't exist, and therefore use the fallback `self:custom.logLevel.default` and set `LOG_LEVEL` to `DEBUG` in that case.

This is a nice trick to specify a stage-specific override, but then fall back to some default value otherwise.

</p></details>
