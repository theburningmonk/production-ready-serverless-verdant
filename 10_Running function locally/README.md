# Module 10: Running function locally

If you run

`npm run sls -- invoke local -f get-restaurants -s dev -r us-east-1`

it'll still exercise the function locally and you will see the following as output

```json
{
    "statusCode": 200,
    "body": "[{\"name\":\"Fangtasia\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/fangtasia.png\",\"themes\":[\"true blood\"]},{\"name\":\"Shoney's\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/shoney's.png\",\"themes\":[\"cartoon\",\"rick and morty\"]},{\"name\":\"Freddy's BBQ Joint\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/freddy's+bbq+joint.png\",\"themes\":[\"netflix\",\"house of cards\"]},{\"name\":\"Pizza Planet\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/pizza+planet.png\",\"themes\":[\"netflix\",\"toy story\"]},{\"name\":\"Leaky Cauldron\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/leaky+cauldron.png\",\"themes\":[\"movie\",\"harry potter\"]},{\"name\":\"Lil' Bits\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/lil+bits.png\",\"themes\":[\"cartoon\",\"rick and morty\"]},{\"name\":\"Fancy Eats\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/fancy+eats.png\",\"themes\":[\"cartoon\",\"rick and morty\"]},{\"name\":\"Don Cuco\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/don%20cuco.png\",\"themes\":[\"cartoon\",\"rick and morty\"]}]"
}
```

You can also run

`npm run sls -- invoke local -f get-restaurants -s dev -r us-east-1 --docker`

which will execute the function locally inside a Docker container, giving you a more realistic execution environment to the Lambda runtime.