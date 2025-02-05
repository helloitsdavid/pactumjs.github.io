# Mock Server

Mock Server allows you to mock any server or service via HTTP or HTTPS, such as a REST endpoint. Simply it is a simulator for HTTP-based APIs.

At one end **pactum** is a REST API testing tool and on the other, it can act as a standalone mock server. It comes in handy while using this library for component & contract testing.

- Simple
- Flexible API
- Powerful Matchers
- Can be Controlled Remotely
- Simulate Faults

## Getting Started

### Basic Setup

The below code will run the pactum mock server on port 3000

<!-- tabs:start -->

#### ** server.js **

```js
const { mock } = require('pactum');
mock.start(3000);
```

<!-- tabs:end -->

Running the mock server.

```shell
node server.js
```

Check health

```shell
# Returns OK
curl http://localhost:3000/api/pactum/health
```

### Start Mock Server

Use `start` to start the mock server on default port 9393 or send a port number as a parameter. It returns a promise. *Waiting for the server to start will be useful while running the mock server along with your unit or component tests.*

```js
// starts mock server on port 9393
await mock.start();
```

```js
// starts mock server on port 3000
await mock.start(3000);
```

### Stop Mock Server

Use `stop` to stop the mock server. It returns a promise. *Waiting for the server to stop will be useful while running the mock server along with your unit or component tests.*

```js
await mock.stop();
```

## Adding Behavior

In general, the role of a mock server is to simulate behavior of a real server. Interactions will help us to simulate the HTTP requests & responses. An interaction contains request & response details. When a real request is sent to mock server, it will try to match the received request with the interactions request. If a match is found it will return the specified response or 404 will be returned.

### Interactions

As said earlier, the simplest way to add behavior to the mock server is through interactions. Use `addInteraction` to add a interaction to the server. It has `request` & `response` objects to handle a request. When the mock server receives a request, it will match the request with interactions `request` properties. If the request matches, the mock server will respond with details in `response` object.

Request Matching:

* Performs *exact match* on received HTTP Method & Path.
* Performs *strong match* on the received request's query params & JSON body.
* Performs *loose match* on received headers.
* Performs *loose match* on the received request's query params & JSON body when **strict** is `false`.

`addInteraction` returns a string containing the id of the interaction. (*This id will be useful when you want to add assertions based on the call count or to check whether the interaction is exercised or not.*)

#### Interaction Options

| Property                  | Description                                 |
| ------------------------  | ------------------------------------------  |
| id                        | id of the interaction                       |
| strict                    | enable/disable strict matching              |
| provider                  | name of the provider                        |
| flow                      | name of the flow                            |
| background                | is a background call                        |
| request                   | request details                             |
| request.method            | HTTP method                                 |
| request.path              | api path                                    |
| request.pathParams        | api path params                             |
| request.headers           | request headers                             |
| request.queryParams       | query parameters                            |
| request.body              | request body                                |
| request.graphQL           | graphQL details                             |
| request.graphQL.query     | graphQL query                               |
| request.graphQL.variables | graphQL variables                           |
| response                  | response details                            |
| response.status           | response status code                        |
| response.headers          | response headers                            |
| response.body             | response body                               |
| response.fixedDelay       | delays the response by ms                   |
| response.randomDelay      | random delay details                        |
| response.randomDelay.min  | delay the response by min ms                |
| response.randomDelay.max  | delay the response by max ms                |
| response.onCall           | response on consecutive calls               |
| response(req, res)        | response with custom function               |
| expects                   | expectations are used in component testing  |
| expects.disable           | disable checks exercised and call count (default: `false`)           |
| expects.exercised         | check exercised (default: `true`)           |
| expects.callCount         | check call count (default: `> 0`)           |
| stores                    | stores data from the request                |

#### Basic Interaction

```js
const { mock } = require('pactum');

mock.addInteraction({
  request: {
    method: 'GET',
    path: '/api/hello'
  },
  response: {
    status: 200,
    body: 'Hello, 👋'
  }
});

mock.start(3000);
```

Open a browser or postman app & opening the following url `http://localhost:3000/api/hello` will respond with a status code `200` & a body with text `Hello, 👋`

Note on how the mock server performs an exact match on HTTP method & path. If a match is not found it will return a status code 404.

- **GET** `http://localhost:3000/api/hello` will return a status code 200.
- **POST** `http://localhost:3000/api/hello` will return a status code 404.

#### Interaction - With Query Params

Let's see an example to send different responses based on query params.

```js
mock.addInteraction({
  request: {
    method: 'GET',
    path: '/api/users',
    queryParams: {
      id: 1
    }
  },
  response: {
    status: 200,
    body: {
      id: 1,
      name: 'Jon'
    }
  }
});

mock.addInteraction({
  request: {
    method: 'GET',
    path: '/api/users',
    queryParams: {
      id: 2
    }
  },
  response: {
    status: 200,
    body: {
      id: 2,
      name: 'Snow'
    }
  }
});
```

- **GET** `/api/users?id=1` will return a status code 200 with **first** user details.
- **GET** `/api/users?id=2` will return a status code 200 with **second** user details.
- **GET** `/api/users` will return a status code 404.
- **GET** `/api/users?id=3` will return a status code 404.
- **GET** `/api/users?name=Jon` will return a status code 404.
- **GET** `/api/users?id=1&name=snow` will return a status code 404.


When `strict` is `false`, it performs a loose match on query params.

```js
mock.addInteraction({
  strict: false,
  request: {
    method: 'GET',
    path: '/api/users',
    queryParams: {
      id: 1
    }
  },
  response: {
    status: 200,
    body: {
      id: 1,
      name: 'Jon'
    }
  }
});
```

- **GET** `/api/users?id=1` will return a status code 200 with **first** user details.
- **GET** `/api/users?id=1&name=snow` will still return a status code 200 with **first** user details.
- **GET** `/api/users?id=1&foo=boo` will still a status code 200 with **first** user details.
- **GET** `/api/users?id=3` will return a status code 404.

#### Interaction - With Body

When `strict` is not mentioned, it performs a strong match on body.

```js
mock.addInteraction({
  request: {
    method: 'POST',
    path: '/api/users',
    body: {
      id: 3,
      married: false,
      country: 'True North'
    }
  },
  response: {
    status: 200
  }
})
```

Posting the below JSON to `/api/users` will return a status code 200.

```json
{
  "id": 3,
  "married": false,
  "country": "True North"
}
```

Posting the below JSON to `/api/users` will return a status code 404. Because it has an additional field `name`.

```json
{
  "id": 4,
  "name": "Jon",
  "married": false,
  "house": "Stark"
}
```

When `strict` is `false`, it performs a loose match on body.

Let's look at an example

```js
mock.addInteraction({
  strict: false,
  request: {
    method: 'POST',
    path: '/api/users',
    body: {
      id: 3,
      married: false,
      country: 'True North'
    }
  },
  response: {
    status: 200
  }
})
```

Posting the below JSON to `/api/users` will return a 200 response.

```json
{
  "id": 3,
  "name": "Jon",
  "married": false,
  "house": "Stark",
  "country": "True North"
}
```

Posting the below JSON to `/api/users` will return a 404 response. The *id* & *country* fields won't match with the above interaction.

```json
{
  "id": 4,
  "name": "Jon",
  "married": false,
  "house": "Stark"
}
```

#### Interaction - With Path Params

Performs a strong match on path params.

```js
mock.addInteraction({
  request: {
    method: 'GET',
    path: '/api/users/{userId}',
    pathParams: {
      userId: '1'
    }
  },
  response: {
    status: 200,
    body: {
      id: 1,
      name: 'Jon'
    }
  }
});
```

#### Behavior on consecutive calls

`onCall` property in `response` defines the behavior of the interaction on the *nth* call. Useful for testing sequential interactions.

```js
mock.addInteraction({
  request: {
    method: 'GET',
    path: '/api/health'
  },
  response: {
    onCall: {
      0: {
        status: 500
      },
      1: {
        status: 200,
        body: 'OK'
      }
    }
  }
})
```

- Invoking **GET** `/api/health` will return a status code 500. (*first call* )
- Invoking **GET** `/api/health` again will return a status code 200. (*second call* )
- Invoking **GET** `/api/health` again will return a status code 404. (*third call* )

Note how the behavior of interaction falls back to the default behavior (*404*) on the third call.

We can also define the default behavior by providing normal response details along with `onCall` details.

```js
response : {
  status: 200,
  onCall: {
    0: {
      status: 500
    }
  }
}
```

- Invoking **GET** `/api/health` will return a status code 500. (*first call* )
- Invoking **GET** `/api/health` again will return a status code 200. (*second call* )
- Invoking **GET** `/api/health` again will return a status code 200. (*third call* )

#### Delays

`fixedDelay` & `randomDelay` adds delay to the response. The following interaction will send a response after *1000 ms*.

```js
mock.addInteraction({
  request: {
    method: 'POST',
    path: '/api/users',
    body: {
      id: 3
    }
  },
  response: {
    status: 200,
    fixedDelay: 1000
  }
})
```

#### Stateful Behavior

Stateful Behavior feature can save us a lot of time when implementing integration test scenarios where we need to send a dynamic response based on the received request. Interactions leverage the features from [Data Management](data-management) to support stateful behavior.

Use `stores` property to capture parts of the request & use it in the response.

For example, consider the mock server receives multiple requests for different projects. Every time we need to pass the received project-id to the response body.

```js
const { mock } = require('pactum');
const { like } = require('pactum-matchers');

mock.addInteraction({
  request: {
    method: 'GET',
    path: '/api/projects/{id}',
    pathParams: {
      id: like('random-id')
    }
  },
  stores: {
    ProjectId: 'req.pathParams.id'
  },
  response: {
    status: 200,
    body: {
      id: '$S{ProjectId}'
    }
  }
});
```

For capturing other parts of the request,

- `req.pathParams` - Request Path Params
- `req.queryParams` - Request Query Params
- `req.headers` - Request Headers
- `req.body` - Request Body

### Handlers

Handlers is a powerful concept in **pactum**. It helps us to reuse different features in this library like expectations, assertions, retry mechanisms, data and many more. Handlers also support interactions.

Interaction handlers help us to reuse same kind of interactions with or without modifications. In **component testing**, interaction handlers play a vital role in re using interactions across different test cases.

`handler.addInteractionHandler` function defines an interaction. The defined interaction can be used later by referring it with its name.

The function accepts two arguments

- handler name - a string to refer the interaction later
- callback function - it should return a interaction/interactions or reference to other interaction/interactions

#### Simple Interaction Handler

```js
const { mock, handler } = require('pactum');

// define interaction 
handler.addInteractionHandler('get users from user-service', () => {
  return {
    request: {
      method: 'GET',
      path: '/api/users'
    },
    response: {
      status: 200,
      body: []
    }
  }    
});

// add the defined interaction to the mock server
mock.addInteraction('get users from user-service');

mock.start(3000);
```

#### Interaction Handler - With Context

We can pass custom data to the interaction handler callback function through the context object which helps us to reuse the interaction with customizations.

The `ctx` object contains a `data` property that will represent the passed custom data.

```js
const { mock, handler } = require('pactum');

handler.addInteractionHandler('get product', (ctx) => {
  return {
    request: {
      method: 'GET',
      path: '/api/inventory',
      queryParams: {
        product: ctx.data.product
      }
    },
    response: {
      status: 200,
      body: {
        "InStock": ctx.data.inStock
      }
    }
  }    
});

mock.addInteraction('get product', { product: 'iPhone', inStock: true });
mock.addInteraction('get product', { product: 'Galaxy', inStock: false });

mock.start(3000);
```

#### Interaction Handler - Refer Other Handlers

Interaction handlers can refer other interaction handlers to make them more reusable. Instead of returning interaction, return an object with `name` & optional `data` property.

```js
const { mock, handler } = require('pactum');

handler.addInteractionHandler('get product', (ctx) => {
  return {
    request: {
      method: 'GET',
      path: '/api/inventory',
      queryParams: {
        product: ctx.data.product
      }
    },
    response: {
      status: 200,
      body: {
        "InStock": ctx.data.inStock
      }
    }
  }    
});

handler.addInteractionHandler('get product in stock', () => {
  // reuses the above interaction
  return { name: 'get product', data: { product: 'iPhone', inStock: true } };   
});

handler.addInteractionHandler('get product out of stock', () => {
  // reuses the first interaction
  return { name: 'get product', data: { product: 'iPhone', inStock: false } };   
});

mock.addInteraction('get product in stock');
mock.addInteraction('get product out of stock');

mock.start(3000);
```

## Matching

Allows matching of request/response with a set of matchers. See [Matching](#matching) for more usage details.

Matchers can be applied to

* Path
* Query Params
* Headers
* Body (*JSON*)

### Type Matching

Often, you will not care what the exact value is at a particular path is, you just care that a value is present and that it is of the expected type.

* `like`
* `eachLike`

#### like

Type matching for primitive data types - *string*/*number*/*boolean*

```js
const { like } = require('pactum-matchers');
const interaction = {
  request: {
    method: 'GET',
    path: '/api/orders',
    queryParams: {
      // matches if it has 'id' property & it is of type string
      id: like('abc')
    }
  },
  response: {
    status: 200,
    body: {
      // Note: Response Matching is used in Contract Testing
      // matches if it has 'quantity' & 'active' properties with number & bool types
      quantity: like(2),
      active: like(true)
    }
  }
}
```

Type matching for objects in **pactum** deviates from in **pact.io** when matching nested objects.

```js
const { like } = require('pactum-matchers');
const interaction = {
  request: {
    method: 'POST',
    path: '/api/orders',
    // matches if the body is JSON 
    // & it has 'quantity' & 'active' properties 
    // & with number & boolean types respectively
    body: like({
      quantity: 2,
      active: true
    })
  },
  response: {
    status: 200
  }
}
```

```js
// matching doesn't expand to nested objects
const { like } = require('pactum-matchers');
const interaction = {
  request: {
    method: 'GET',
    path: '/api/orders'
  },
  response: {
    status: 200,
    // Note: Response Matching is used in Contract Testing
    // matches if body is JSON object
    // & it has 'quantity', 'active' & 'item' properties 
    // & with number, bool & object types respectively
    // & item has only 'name' & 'brand' properties with 'car' & 'v40' values respectively
    body: like({
      quantity: 2,
      active: true,
      item: {
        name: 'car',
        brand: 'v40'
      }
    })
  }
}
```

```js
// to match nested objects with type, we need apply 'like()' explicitly to nested objects
const { like } = require('pactum-matchers');
const interaction = {
  request: {
    method: 'POST',
    path: '/api/orders',
    // matches if body is JSON object
    // & it has quantity, active, item & delivery properties 
    // & with number, bool & object types respectively
    // & item has name & brand properties with string types
    // & delivery has pin property with type number 
    // & delivery has state property with 'AP' as value
    body: like({
      quantity: 2,
      active: true,
      item: like({
        name: 'car',
        brand: 'v40'
      }),
      delivery: {
        pin: like(500081),
        state: 'AP'
      }
    })
  },
  response: {
    status: 200
  }
}
```

#### eachLike

*eachLike* is similar to *like* but applies to arrays.

```js
const { eachLike } = require('pactum-matchers');
const interaction = {
  request: {
    method: 'GET',
    path: '/api/orders'
  },
  response: {
    status: 200,
    // Note: Response Matching is used in Contract Testing
    // matches if body is an array 
    // & each item in the array is an object
    // & each object should have quantity, active, product properties
    // & quantity, active, product should be of types number, boolean & object
    // & product has a name property with string type
    // & product has a colors property with array of strings
    body: eachLike({
      quantity: 2,
      active: true,
      product: like({
        name: 'car',
        colors: eachLike('blue')
      }
    })
  }
}
```

### Regex Matching

Sometimes you will have keys in a request or response with values that are hard to know beforehand - timestamps and generated IDs are two examples.

What you need is a way to say "I expect something matching this regular expression, but I don't care what the actual value is".

#### regex

`regex` accepts a value and a matcher as arguments.

```js
const { regex } = require('pactum-matchers');
const interaction = {
  request: {
    method: 'POST',
    path: '/api/projects/12',
    body: {
      name: 'Jon'
      birthDate: regex('03/05/2020', /\d{2}\/\d{2}\/\d{4}/)
    }
  },
  response: {
    status: 200
  }
}
```

## Remote API

Pactum allows us to add or remove interactions dynamically through the REST API.
Once the server is started, interact with the following APIs to control the mock server.

```js
const mock = require('pactum').mock;
mock.start(3000);
```

### Interactions

#### /api/pactum/interactions

##### GET

Returns a single or all interactions.

```Shell
# Fetches a interaction with id "m1uh9"
curl --location --request GET 'http://localhost:9393/api/pactum/interactions?id=m1uh9'

# Fetches all interactions
curl --location --request GET 'http://localhost:9393/api/pactum/interactions'
```

Response - returns all the interactions

```JSON
[
  {
    "id": "<id>",
    "strict": true,
    "exercised": false,
    "calls": [],
    "callCount": 0,
    "request": {
      "matchingRules": {},
      "method": "GET",
      "path": "/api/projects/2",
      "query": {
        "name": "fake"
      }
    },
    "response": {
      "matchingRules": {},
      "status": 200,
      "headers": {
        "content-type": "application/json"
      },
      "body": {
        "id": 1,
        "name": "fake"
      }
    },
    "expects": {
      "disable": false,
      "exercised": true
    }
  }
]
```

##### POST

Adds multiple interactions to the server.

```Shell
curl --location --request POST 'http://localhost:9393/api/pactum/interactions' \
--header 'Content-Type: application/json' \
--data-raw '[{
    "request": {
        "method": "GET",
        "path": "/api/projects/2",
        "query": {
            "name": "fake"
        }
    },
    "response": {
        "status": 200,
        "headers": {
            "content-type": "application/json"
        },
        "body": {
            "id": 1,
            "name": "fake"
        }
    }
  }]'
```

Response - returns the interaction id.

```JSON
[ "x121w" ] 
```

##### DELETE

Removes a interaction or all interactions from the mock server.

```Shell
# Removes a single interaction with id m1uh9
curl --location --request DELETE 'http://localhost:9393/api/pactum/interactions?id=m1uh9'

# Removes all interactions
curl --location --request DELETE 'http://localhost:9393/api/pactum/interactions'
```

### Interaction Handlers

Interaction Handler allows us to dynamically enable & reuse interactions in the mock server. Let's say you have a mock server running on port 3000 with one interaction handler.

```js
const { mock, handler } = require('pactum');

handler.addInteractionHandler('get user with', (ctx) => {
  const user = db.getUser(ctx.data.id);
  return {
    request: {
      method: 'GET',
      path: `/api/users/${ctx.data.id}`
    },
    response: {
      status: 200,
      body: user
    }
  }
});

mock.start(3000);
```

If you start the above mock server, by default it will have zero interactions in it. To add the interaction with handler name `'get user with'`, perform the following request.

```Shell
curl --location --request POST 'http://localhost:9393/api/pactum/handlers' \
--header 'Content-Type: application/json' \
--data-raw '[ { "name": "get user with", "data": { "id": "some-random-id" } } ]'
```

Response - returns the interaction id.

```JSON
[ "x121w" ]
```

### State

State handlers helps us to run specific asynchronous code that puts our application in a specific state.

```js
const { mock, handler } = require('pactum');

handler.addStateHandler('some state name', async (ctx) => {
  const { data } = ctx;
  // code to add data in database - redis.set()
  // or code to add mock interactions - mock.addInteraction()
  // or custom code
});

mock.start(3000);
```

Run the state handler using

```Shell
curl --location --request POST 'http://localhost:9393/api/pactum/state' \
--header 'Content-Type: application/json' \
--data-raw '[ { "name": "some state name", "data": { "id": "some-random-id" } } ]'
```
