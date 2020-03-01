# Module 20: Sample debug logs in production

## Sample 1% of debug logs in production

<details>
<summary><b>Install lambda-powertools-pattern-basic</b></summary><p>

1. At the project root, run the command `npm install --save @dazn/lambda-powertools-pattern-basic`.

This package gives you a simple wrapper which applies a couple of [middy](https://github.com/middyjs/middy) middlewares for your function:

* `@dazn/lambda-powertools-middleware-sample-logging`: which supports sampling debug logs. The wrapper configures this sample logging middleware to sample debug logs for 1% of invocations.

* `@dazn/lambda-powertools-middleware-correlation-ids`: which extracts correlation IDs from the invocation event and makes them available for the logger. It also supports a special correlation ID `debug-log-enabled`, which enables sampling debug logs at the user transaction (a chain of Lambda invocations) level.

* `@dazn/lambda-powertools-middleware-log-timeout`: which emits an error message for when a function times out. Normally, when a Lambda function times out, you don't get an error message from the application, which makes debugging time out errors difficult.

Now we need to apply it to all of our functions.

</p></details>

<details>
<summary><b>Wrap function handlers</b></summary><p>

1. Modify `functions/get-index.js` to require the `@dazn/lambda-powertools-pattern-basic` module (at the top of the file)

```javascript
const wrap = require('@dazn/lambda-powertools-pattern-basic')
```

And use it to wrap our handler function. Change `module.exports.handler = async (event, context) => {` to the following (don't forget the closing `)` at the end!)

```javascript
module.exports.handler = wrap(async (event, context) => {
  ...
})
```

2. Repeat step 1 for **all the function handlers**.

3. Run integration test

`STAGE=dev REGION=us-east-1 npm run test`

and see that tests are failing with error messages like this:

```
1) When we invoke the GET / endpoint
       Should return the index page with 8 restaurants:
     TypeError: callback is not a function
      at terminate (node_modules/middy/src/middy.js:152:16)
      at runNext (node_modules/middy/src/middy.js:126:14)
      at runErrorMiddlewares (node_modules/middy/src/middy.js:130:3)
      at errorHandler (node_modules/middy/src/middy.js:160:14)
      at runMiddlewares (node_modules/middy/src/middy.js:164:23)
      at runNext (node_modules/middy/src/middy.js:87:14)
      at runMiddlewares (node_modules/middy/src/middy.js:91:3)
      at instance (node_modules/middy/src/middy.js:163:5)
      at viaHandler (tests/steps/when.js:82:26)
      at Object.we_invoke_get_index (tests/steps/when.js:93:15)
      at Context.it (tests/test_cases/get-index.js:10:28)
      at process.topLevelDomainCallback (domain.js:120:23)
```

This is because, `middy` turns our functions into callback style functions so that it's backward compatible with Node 6.10 as well.

So we need to update `tests/steps/when.js` to match this. We'll use `util.promisify` to turn the handler function back to async function.

4. Modify `tests/steps/when.js` to use `util.promisify` to turn the handler function back to async function

First, require the `util` module at the top.

```javascript
const util = require('util')
```

Then replace the `viaHandler` function with the following

```javascript
const viaHandler = async (event, functionName) => {
  const handler = util.promisify(require(`${APP_ROOT}/functions/${functionName}`).handler)
  console.log(`invoking via handler function ${functionName}`)

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (_.get(response, 'body') && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

5. Rerun integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all the tests are now passing

6. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

</p></details>
