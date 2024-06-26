# jest-fetch-mock-cache

Caching mock implementation for jest-fetch-mock.

Copyright (c) 2023 by Gadi Cohen. [MIT Licensed](./LICENSE.txt).

![npm](https://img.shields.io/npm/v/jest-fetch-mock-cache) ![GitHub Workflow Status (with event)](https://img.shields.io/github/actions/workflow/status/gadicc/jest-fetch-mock-cache/release.yml) ![Coverage](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/gadicc/26d0f88b04b6883e1a6bba5b9b344fab/raw/jest-coverage-comment__main.json) [![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release) [![TypeScript](https://img.shields.io/badge/%3C%2F%3E-TypeScript-%230074c1.svg)](http://www.typescriptlang.org/) [![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)

## Introduction

Instead of individually handcrafting a mock for each and every `fetch()`
call in your code, maybe you'd like to perform a real `fetch()` once,
cache the result, and use that cache result for future calls. **Super
useful for TDD against existing APIs!!**

Note: this is a brand new project. It works but you may want to wait
a little before any serious use. Feature requests welcome!

## Quick Start

```ts
import fetchMock from "jest-fetch-mock";
fetchMock.enableMocks();

import { describe, expect, test as it } from "@jest/globals";
import createCachingMock from "jest-fetch-mock-cache";

// See list of possible stores, below.
import Store from "jest-fetch-mock-cache/lib/stores/nodeFs";

const cachingMock = createCachingMock({ store: new Store() });

describe("cachingMock", () => {
  it("works with a JSON response", async () => {
    const url = "http://echo.jsontest.com/key/value/one/two";
    const expectedResponse = { one: "two", key: "value" };

    for (let i = 0; i < 2; i++) {
      // If you get a TypeError here, make sure @types/jest is instaled.
      fetchMock.mockImplementationOnce(cachingMock);
      const response = await fetch(url);
      const data = await response.json();
      const expectedCacheHeader = i === 0 ? "MISS" : "HIT";
      expect(response.headers.get("X-JFMC-Cache")).toBe(expectedCacheHeader);
      expect(data).toEqual(expectedResponse);
    }
  });
});
```

- The **first** time this runs, a _real_ request will be made to
  `jsontest.com`, and the result returned. But, it will also be
  saved to cache.

- **Subsequent requests** will return the cached copy without
  making an HTTP request.

Note: the package is designed to be used with `jest-fetch-mock`, but that's
not required. Since it's a complete impleemntation on its own, you could
do something like

```ts
// Do this and then use this `fetch()` function, or set global.fetch = ...
const fetch = jest.fn(createCachingMock({ store: new Store() }));
```

## What's cached

Sample output from the Quick Start code above, when used with `NodeFSStore`:

```bash
$ cat tests/fixtures/http/echo.jsontest.com\!key\!value\!one\!two
```

```js
{
  "request": {
    "url": "http://echo.jsontest.com/key/value/one/two"
  },
  "response": {
    "ok": true,
    "status": 200,
    "statusText": "OK",
    "headers": {
      "access-control-allow-origin": "*",
      "connection": "close",
      "content-length": "39",
      "content-type": "application/json",
      "date": "Fri, 21 Jul 2023 16:59:17 GMT",
      "server": "Google Frontend",
      "x-cloud-trace-context": "344994371e51195ae21f236e5d7650c4"
    },
    "bodyJson": {
      "one": "two",
      "key": "value"
    }
  }
}
```

For non-JSON bodies, a `bodyText` is stored as a string. We store an
object as `bodyJson` for readability reasons.

## Debugging

We use [debug](https://www.npmjs.com/package/debug) for debugging. E.g.:

```bash
$ DEBUG=jest-fetch-mock-cache:* yarn test
yarn run v1.22.19
$ jest
  jest-fetch-mock-cache:core [jsmc] Fetching and caching 'http://echo.jsontest.com/key/value/one/two' +0ms
  jest-fetch-mock-cache:core [jsmc] Using cached copy of 'http://echo.jsontest.com/key/value/one/two' +177ms
 PASS  src/index.spec.ts
  cachingMock
    ✓ should work (180 ms)
```

## Available Stores

```ts
import createCachingMock from "jest-fetch-mock-cache";

// Use Node's FS API to store as (creating paths as necessary)
// `${process.cwd()}/tests/fixtures/${filenamifyUrl(url)}`.
// https://github.com/sindresorhus/filenamify-url
import JFMCNodeFSStore from "./stores/nodeFs";
const fsCacheMock = createCachingMock({ store: new JFMCNodeFSStore() });
fetchMock.mockImplementationOnce(fsCacheMock);
// To override the store location, init with store with e.g.:
// new JFMCNodeFSStore({ location: "tests/other/location" })

// Keep in memory
import JSMCMemoryStore from "./stores/memory";
const memoryCacheMock = createCachingMock({ store: new JSMCMemoryStore() });
fetchMock.mockImplementationOnce(memoryCacheMock);
```

### Create your own Store

```ts
import JFMCStore from "jest-fetch-mock-cache/lib/store";
import type { JFMCCacheContent } from "jest-fetch-mock-cache/lib/store";
import db from "./db"; // your existing db

class MyCustomStore extends JFMCStore {
  async fetchContent(req: Request): Promise<JFMCCacheContent | undefined> {
    const key = await this.idFromResponse(req);
    return (await db.collection("jfmc").findOne(key)).content;
  }

  async storeContent(req: Request, content: JFMCCacheContent): Promise<void> {
    const key = await this.idFromResponse(req);
    await db.collection("jfmc").insertOne({ _id: key, content });
  }
}

export default MyCustomStore;
```

## TODO

- [ ] Browser-environment support. Please open an issue if you need this, and in what cases. jsdom?
- [x] Cache request headers too and hash them in filename / key / id.
- [ ] Handle and store invalid JSON too.
