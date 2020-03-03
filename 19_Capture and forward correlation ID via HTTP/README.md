# Module 19: Capture and forward correlation ID via HTTP

## Capture and forward correlation IDs

Since we're using the `@dazn/lambda-powertools-pattern-basic` and `@dazn/lambda-powertools-logger`, we are already capturing incoming correlation IDs and including them in the logs.

However, we need to make sure the `get-index` function forwards the correlation IDs along, while still using the X-Ray instrumented `https` module.

The `dazn-lambda-powertools` project also has Kinesis and SNS clients that will automatically forward captured correlation IDs. We can use them as stand-in replacements for the Kinesis and SNS clients from the AWS SDK.

<details>
<summary><b>Forward correlation IDs from get-index</b></summary><p>

You can access the auto-captured correlation IDs using the `@dazn/lambda-powertools-correlation-ids` package. From here, we can include them as HTTP headers.

1. At the project root, run `npm install --save @dazn/lambda-powertools-correlation-ids`.

2. Open `functions/get-index.js` and require the `@dazn/lambda-powertools-correlation-ids` module (at the top of the file).

```javascript
const CorrelationIds = require('@dazn/lambda-powertools-correlation-ids')
```

3. Staying in `functions/get-index.js`, replace the `getRestaurants` function with the following

```javascript
const getRestaurants = () => {
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname, 
    path: url.pathname
  }

  aws4.sign(opts)

  return new Promise((resolve, reject) => {
    const options = {
      hostname: url.hostname,
      port: 443,
      path: url.pathname,
      method: 'GET',
      headers: Object.assign({}, opts.headers, CorrelationIds.get())
    }

    const req = https.request(options, res => {
      res.on('data', buffer => {
        const body = buffer.toString('utf8')
        resolve(JSON.parse(body))
      })
    })

    req.on('error', err => reject(err))

    req.end()
  })
}
```

These changes are enough to ensure correlation IDs are included in the HTTP headers in the request to the `GET /restaurants` endpoint.

4. Open `tests/steps/when.js` and modify the `viaHandler` method so that `context` is initialized with a `awsRequestId`, e.g.

```
const context = { awsRequestId: 'test' }
```

This will be used to initialize the correlation ID with and you'll see it in the logs when you run the test

`STAGE=dev REGION=us-east-1 npm run test`

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
{"message":"loading index.html...","awsRegion":"us-east-1","environment":"dev","awsRequestId":"test","x-correlation-id":"test","debug-log-enabled":"false","call-chain-length":1,"level":30,"sLevel":"INFO"}
{"message":"loaded","awsRegion":"us-east-1","environment":"dev","awsRequestId":"test","x-correlation-id":"test","debug-log-enabled":"false","call-chain-length":1,"level":30,"sLevel":"INFO"}
    ✓ Should return the index page with 8 restaurants (1664ms)
    
  ...
```

5. Open `functions/get-restaurants.js`, require the `@dazn/lambda-powertools-logger` logger at the top

`const Log = require('@dazn/lambda-powertools-logger')`

and then add a single debug log message at the start of the handler, e.g.

```
module.exports.handler = wrap(async (event, context) => {
  Log.debug('loading restaurants')
  ...
  
}
```

6. Redeploy the project.

`npm run sls -- deploy -s dev -r us-east-1`

7. Once the deployment is done, load the page. And then open the X-Ray console to make sure that the X-Ray tracing is still working.

8. Open the CloudWatch console to check the logs for both `get-index` and `get-restaurants`. You should see that the same correlation ID is included in both logs.

</p></details>

<details>
<summary><b>Forward correlation IDs through Kinesis events</b></summary><p>

As an exercise for you to carry out on your own after the workshop. See if you can get correlation IDs flowing through the Kinesis events as well.

Familiarize yourself with the various powertools in the dazn-lambda-powertools [repo](https://github.com/getndazn/dazn-lambda-powertools). You will need:

* the [Kinesis client](https://github.com/getndazn/dazn-lambda-powertools/tree/master/packages/lambda-powertools-kinesis-client) as a stand-in replacement for the AWS SDK's Kinesis client. This client would auto-forward any captured correlation IDs along.

* for batched event sources (Kinesis, Firehose and SQS), the [correlation-IDs middleware](https://github.com/getndazn/dazn-lambda-powertools/tree/master/packages/lambda-powertools-middleware-correlation-ids), which is applied through the `wrap` function we used to wrap our handlers, requires you to change your processor code slightly. Read the [relevant section](https://github.com/getndazn/dazn-lambda-powertools/tree/master/packages/lambda-powertools-middleware-correlation-ids#kinesis) of the README and change the handler code accordingly.

Also, check out [**this repo**](https://github.com/theburningmonk/lambda-distributed-tracing-demo/tree/master/lambda-powertools) to see a more comprehensive demo of how you can auto-extract and forward correlation IDs through a variety of different event sources with the dazn-lambda-powertools.

</p></details>
