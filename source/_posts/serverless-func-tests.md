---
title: Implement functional tests for serverless architectures
date: 2019-06-19 12:32:45
tags:
- NodeJS
- Typescript
- Serverless
- AWS
categories: 
- Tech
---

# Implement functional tests for serverless architectures

I recently have to write functional tests for a NodeJS serverless application.

The setup process is not that obvious so I decided to post an article here to explain how to get started.

## Set up typescript tests using jest

Functional tests are run using [jest](https://jestjs.io).

For maintainabilty reasons (and because I love it), I will use typescript to write my tests.

1. Install the following dependencies in your project:

```bash
npm i -D jest @types/jest ts-jest
```

2. In your package.json add the following scripts (I assumed you also have, or want, unit tests)

```json
{
  "scripts": {
    "test": "npm run test:unit && npm run test:functional",
    "test:unit": "echo \"Info: no unit tests here\" && exit 0",
    "test:functional": "NODE_ENV=test jest --config jest.config.functional.json"
  }
}
```

3. And create the following config file to get started with typescripts tests:

```json
{
  "testEnvironment": "node",
  "roots": [
    "<rootDir>/test"
  ],
  "transform": {
    "^.+\\.tsx?$": "ts-jest"
  },
  "maxConcurrency": 5,
  "testRegex": ".+\\.(test|spec)\\.tsx?$",
  "moduleFileExtensions": [
    "ts",
    "tsx",
    "js",
    "jsx",
    "json",
    "node"
  ],
  "moduleNameMapper": {
    "@/(.*)": "<rootDir>/src/$1"
  }
}
```

Jest will look for test files in `test` directory ending with `*.spec.ts`. 

## Spawn serverless offline

Before the functional tests are run, we must start serverless offline on a given port.

We use jest `globalSetup` and `globalTeardown` hooks to start and shutdown the serverless offline server only once.

Since jest v24, it is now possible to implement these hooks directly in typescript and use them without pre-compile steps.

In your `jest.config.functional.json` configuration file add:

```json
{
  "globalSetup": "<rootDir>/test/hooks/setup.ts",
  "globalTeardown": "<rootDir>/test/hooks/teardown.ts"
}
```

Then create the `test/hooks/setup.ts` with the following setup:

```typescript
import { spawnServer } from "../helpers/spawn-server";

export default async function(): Promise<void> {
  console.log('\nStarting serverless offline');
  // @ts-ignore
  global.__offline__ = await spawnServer();
  console.log('Success');
};
```

The `spawnServer` method will spawn `sls offline` in a NodeJS child process.
Here is the content of `test/helpers/spawn-server.ts`

```typescript
import { ChildProcess, spawn } from "child_process";

export const spawnServer = async (host?: string, port?: number): Promise<ChildProcess> => {
  return new Promise((resolve, reject) => {
    const offline = spawn('sls', [
      'offline',
      'start',
      '--host',
      host || 'localhost',
      '--port',
      port ? port.toString() : '4848',
    ]);
    offline.stdout.on('data', (data) => {
      if (data.includes("Offline listening on")) {
        resolve(offline);
      }
    });
    offline.stderr.on('data', (err) => {
      reject(err);
    });
  });
};
```

As you can see, we spawn here a child process to start serverless offline on a given host/port.
You can change these settings if you have multiple serverless "microservices" and you want to run tests concurrently.
To make sure the server is correctly started we watch for *Offline listening on* in the command `stdout` ~~("feels like a hack but it works")~~.

The spawned process is stored in `global.__offline__` variable, which is also available in teardown hook. Thus to stop the server after all tests are executed, simply put this in the teardown hook `test/hooks/teardown.ts`:

```typescript
 export default async function(): Promise<void> {
  console.log('Stopping serverless offline');
  // @ts-ignore
  global.__offline__.kill();
  console.log('SIGTERM sent');
};
```

Now you are ready to write yout first functional test !

## Write your first test

You can use a HTTP client like [got](https://github.com/sindresorhus/got) or a dedicated test library such as [supertest](https://github.com/visionmedia/supertest) to perform request on the serverless offline server you just have spawned.

Here is a very basic exemple:

```typescript
import got from './utils/got';

const req = got.extend({
  baseUrl: 'http://localhost:4848',
  method: 'POST',
});

describe('The order create handler - POST /orders', () => {
  test('should return 422 if product ID is missing', (done) => {
    req('orders', {
      body: JSON.stringify({
        customerId: 'customer@example.com',
      }),
    }).then(() => {
      fail('should not return 200');
    })
    .catch((err) => {
      expect(err.statusCode).toBe(422);
      done();
    });
  });
  test('should return 422 if customer ID is missing', (done) => {
    req('orders', {
      body: JSON.stringify({
        productId: '110e8400-e29b-11d4-a716-446655440000',
      }),
    }).then(() => {
      fail('should not return 200');
    })
    .catch((err) => {
      expect(err.statusCode).toBe(422);
      done();
    });
  });
  test('should return 422 if customer ID is not a valid email', (done) => {
    req('v4/access-requests', {
      body: JSON.stringify({
        productId: '110e8400-e29b-11d4-a716-446655440000',
        customerId: 'invalidCustomer',
      }),
    }).then(() => {
      fail('should not return 200');
    })
    .catch((err) => {
      expect(err.statusCode).toBe(422);
      done();
    });
  });
  test.todo('Should return 400 if product does not exist in database');
  test.todo('Should return 400 if customer does not exist in database');
  test.todo('Should return 200 if everything is OK');
  // etc
});
```

## Use a custom authorizer

If your handler uses a custom authorizer, you need to mock the authorizer lambda return value.
Here is a `serverless.yml` example that impelments such a mechanism.

```yml

functions:
  auth:
    handler: src/handlers/authauth.handler
  order-create:
    handler: src/handlers/orders/create.handler
    events:
      - http:
          path: orders
          authorizer:
            name: auth
            type: request
          method: post
          cors:
            origins:
              - '*'
            headers:
              - Authorization
              - Content-Type
              - X-Amz-Date
              - X-Amz-Security-Token
              - X-Api-Key
              - X-Impersonate
            allowCredentials: true
```

To mock the authorization process, add a "test-auth" function in your yaml.
Then set the real or the mocked authorizer function that is called in your endpoints depending on your `process.env.NODE_ENV`.

```yaml
functions:
  auth:
    handler: src/handlers/auth/auth.handler
  auth-test:
    handler: test/helpers/auth.handler
  access-request-create:
    handler: src/handlers/orders/create.handler
    events:
      - http:
          path: orders
          authorizer:
            name: ${self:custom.authorizerName.${env:ENV}, self.authorizerName.local}
            type: request
          method: post
          cors:
            origins:
              - '*'
            headers:
              - Authorization
              - Content-Type
              - X-Amz-Date
              - X-Amz-Security-Token
              - X-Api-Key
              - X-Impersonate
            allowCredentials: true
custom:
  authorizerName:
    local: auth
    test: auth-test
```

Now you can use this mocked authorizer `test/helpers/auth.handler` to authenticate and authorize your test request.

Here is an example to mock a two-levels (admin/user) authentication process:

```typescript
import { AuthResponse, CustomAuthorizerEvent } from "aws-lambda";

export const mockedAuthHandler = async (event: CustomAuthorizerEvent): Promise<AuthResponse> => {

  const authHeader = getHeader(event, 'authorization');

  const policyDocument = {
    Version: '2012-10-17',
    Statement: [
      {
        Action: 'execute-api:Invoke',
        Effect: 'Allow',
        Resource: '*',
      },
    ],
  };

  const adminId = 'test.admin@example.com';
  const userId = 'test.user@example.com';

  const tokenMatch = authHeader ? authHeader.match(/^Bearer (.+)$/i) : null;
  if (!tokenMatch) {
    throw Error('Unauthorized');
  }
  const token = tokenMatch[1];
  switch(token) {
    case 'admin-token':
      return {
        principalId: adminId,
        policyDocument,
        context: {
          currentUser: JSON.stringify({
            user_id: adminId,
            role: 'admin',
          }),
        },
      };
    case 'user-token':
      return {
        principalId: userId,
        policyDocument,
        context: {
          currentUser: JSON.stringify({
            user_id: userId,
            role: 'user',
          }),
        },
      };
    default:
        throw Error('Unauthorized');
  }
};

const getHeader = (event: CustomAuthorizerEvent, name: string): string => {
  if (!event.headers) {
    return null;
  }
  const headerName = Object.keys(event.headers).find((header) => header.toLowerCase() === name);
  return headerName != null ? event.headers[headerName].toLowerCase() : null;
};
```

We simply authenticate user making the request as admin or non-admin base on two-values mocked token: `user-token` or `admin-token`. Then we format the response to meet AWS custom authorizer requirements.

The real authentication would decode a JWT and perform a database request to provide context and current logged in user.

Now you have to update your functional tests and send the token in your requests:

```typescript
  test('should return 401 if token is missing', (done) => {
    req('orders')
      .then(() => {
        fail('should not return 200');
      })
      .catch((err) => {
        expect(err.statusCode).toBe(401);
        done();
      });
  });
    test('should return 422 if product ID is missing', (done) => {
    req('orders', {
      headers: {
        Authorization: 'Bearer user-token',
      },
      body: JSON.stringify({
        customerId: 'customer@example.com',
      }),
    }).then(() => {
      fail('should not return 200');
    })
    .catch((err) => {
      expect(err.statusCode).toBe(422);
      done();
    });
  });
  // And so on....
```

