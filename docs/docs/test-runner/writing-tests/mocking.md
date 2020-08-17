---
title: Mocking
eleventyNavigation:
  key: Writing tests > Mocking
  title: Mocking
  parent: Writing tests
  order: 50
---

## Mocking functions

For stubbing and mocking functions, we recommend [sinon](https://www.npmjs.com/package/sinon).

Sinon ships an es module, you can import it in your tests like this:

```js
import { stub, spy, useFakeTimers } from 'sinon';
```

## Mocking es modules

Es modules are immutable, it's not possible to mock or stub imports:

```js
import { stub } from 'sinon';
import * as myModule from './my-module.js';

// not possible
stub(myModule, 'myImport');
```

We can use [Import Maps](https://github.com/WICG/import-maps) in our tests to resolve module imports to a mocked version.

### Using Import Maps to tests

To use Import Maps in your tests, you first need to install the plugin. Go to the [Import Maps Plugin](../../dev-server/plugins/import-maps.md) page to learn how to do that.

Import Maps are defined in HTML, we need to write a [HTML test](./html-tests.md) to use them.

This is an example using Mocha. We want to test if a function works correctly without actually causing side effects, like posting data to a server. We mock the module that causes side effects in our test.

Source code:

`/postData.js:`

```js
export function postData(endpoint, data) {
  return fetch(`/api/${endpoint}`, { method: 'POST', body: JSON.stringify(data) });
}
```

`/postMessage.js:`

```js
import { postData } from './postData.js';

export function postMessage(message) {
  return postData('message', { message })`;
}
```

Mocked module:

`/mocks/postData.js:`

```js
import { stub } from 'sinon';

export const postData = stub();
```

Test file:

```html
<html>
  <head>
    <!-- the import map to use in our test -->
    <script type="importmap">
      {
        "imports": {
          "./postData.js": "./mocks/postData.js"
        }
      }
    </script>
  </head>

  <body>
    <script type="module">
      import { runTests } from '@web/test-runner-mocha';
      // import inside will resolve to ./mocks/postData.js
      import { postMessage } from './postMessage.js';
      // resolves to ./mocks/postData.js
      import { postData } from './postData.js';

      runTests(() => {
        describe('postMessage()', () => {
          it('calls postData', () => {
            postMessage('foo');
            expect(postData.callCount).to.equal(1);
            expect(postData.getCall(0).args).to.eql(['message', { message: 'foo' }]);
          });
        });
      });
    </script>
  </body>
</html>
```