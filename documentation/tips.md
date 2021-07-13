[> Back to homepage](../readme.md#documentation)

## Tips

### Timeout

Each request can have a maximum allowed time to run.\
In order to use this, specify the `request` timeout option.

```js
import got from 'got';

const body = await got('https://httpbin.org/anything', {
	timeout: {
		request: 30000
	}
});
```

For more specific timeouts, visit the [Timeout API](6-timeout.md).

### Retries

By default, Got makes a new retry on a failed request if possible.

It is possible to disable this feature entirely by setting the amount of maximum allowed retries to `0`.

```js
import got from 'got';

const noRetryGot = got.extend({
	retry: {
		limit: 0
	}
});
```

In order to specify retriable errors, use the [Retry API](7-retry.md).

### Cookies

Got supports cookies out of box. There is no need to parse them manually.\
In order to use cookies, pass a `CookieJar` instance from the [`tough-cookie`](https://github.com/salesforce/tough-cookie) package.

```js
import {promisify} from 'util';
import got from 'got';
import {CookieJar} from 'tough-cookie';

const cookieJar = new CookieJar();
const setCookie = promisify(cookieJar.setCookie.bind(cookieJar));

await setCookie('foo=bar', 'https://httpbin.org');
await got('https://httpbin.org/anything', {cookieJar});
```

### AWS

Requests to AWS services need to have their headers signed.\
This can be accomplished by using the [`got4aws`](https://github.com/SamVerschueren/got4aws) package.

This is an example for querying an [`API Gateway`](https://docs.aws.amazon.com/apigateway/api-reference/signing-requests/) with a signed request.

```js
import got4aws from 'got4aws';

const got = got4aws();

const response = await got('https://<api-id>.execute-api.<api-region>.amazonaws.com/<stage>/endpoint/path', {
	// …
});
```

### Pagination

When working with large datasets, it's very efficient to use pagination.\
By default, Got uses the [`Link` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Link) to retrieve the next page.\
However, this behavior can be customized, see the [Pagination API](4-pagination.md).

```js
const countLimit = 50;

const pagination = got.paginate('https://api.github.com/repos/sindresorhus/got/commits', {
	pagination: {countLimit}
});

console.log(`Printing latest ${countLimit} Got commits (newest to oldest):`);

for await (const commitData of pagination) {
	console.log(commitData.commit.message);
}
```

<a name="unix"></a>
### UNIX Domain Sockets

Requests can also be sent via [UNIX Domain Sockets](https://serverfault.com/questions/124517/what-is-the-difference-between-unix-sockets-and-tcp-ip-sockets).\
Use the following URL scheme: `PROTOCOL://unix:SOCKET:PATH`

- `PROTOCOL` - `http` or `https`
- `SOCKET` - Absolute path to a unix domain socket, for example: `/var/run/docker.sock`
- `PATH` - Request path, for example: `/v2/keys`

```js
import got from 'got';

await got('http://unix:/var/run/docker.sock:/containers/json');

// Or without protocol (HTTP by default)
await got('unix:/var/run/docker.sock:/containers/json');
```

### Testing

Got uses the native [`http`](https://nodejs.org/api/http.html) module, which depends on the native [`net`](https://nodejs.org/api/net.html) module.\
This means there are two possible ways to test:

1. Use a mocking library like [`nock`](https://github.com/nock/nock),
2. Create a server.

The first approach should cover all common use cases.\
Bear in mind that it overrides the native `http` module, so bugs may occur due to the differences.

The most solid way is to create a server.\
There may be cases where `nock` won't be sufficient or lacks functionality.

#### Nock

By default `nock` mocks only one request.\
Got will [retry](7-retry.md) on failed requests by default, causing a `No match for request ...` error.\
The solution is to either disable retrying (set `options.retry.limit` to `0`) or call `.persist()` on the mocked request.

```js
import got from 'got';
import nock from 'nock';

const scope = nock('https://sindresorhus.com')
	.get('/')
	.reply(500, 'Internal server error')
	.persist();

try {
	await got('https://sindresorhus.com')
} catch (error) {
	console.log(error.response.body);
	//=> 'Internal server error'

	console.log(error.response.retryCount);
	//=> 2
}

scope.persist(false);
```

### Proxying

You can use the [`tunnel`](https://github.com/koichik/node-tunnel) package with the `agent` option to work with proxies:

```js
import got from 'got';
import tunnel from 'tunnel';

await got('https://sindresorhus.com', {
	agent: {
		https: tunnel.httpsOverHttp({
			proxy: {
				host: 'localhost'
			}
		})
	}
});
```

Otherwise, you can use the [`hpagent`](https://github.com/delvedor/hpagent) package, which keeps the internal sockets alive to be reused.

```js
import got from 'got';
import {HttpsProxyAgent} from 'hpagent';

await got('https://sindresorhus.com', {
	agent: {
		https: new HttpsProxyAgent({
			keepAlive: true,
			keepAliveMsecs: 1000,
			maxSockets: 256,
			maxFreeSockets: 256,
			scheduling: 'lifo',
			proxy: 'https://localhost:8080'
		})
	}
});
```

Alternatively, use [`global-agent`](https://github.com/gajus/global-agent) to configure a global proxy for all HTTP/HTTPS traffic in your program.

If you're using HTTP/2, the [`http2-wrapper`](https://github.com/szmarczak/http2-wrapper/#proxy-support) package provides proxy support out-of-box.\
[Learn more.](https://github.com/szmarczak/http2-wrapper#proxy-support)

### Retry without an agent

If you're using proxies, you may run into connection issues.\
One way out is to disable proxies when retrying. The solution for the Stream API looks like this:

```js
import https from 'https';
import got from 'got';

class MyAgent extends https.Agent {
	createConnection(port, options, callback) {
		console.log(`Connecting with MyAgent`);
		return https.Agent.prototype.createConnection.call(this, port, options, callback);
	}
}

const proxy = new MyAgent();

const fn = retryCount => {
	const stream = got.stream('https://httpbin.org/status/408', {
		agent: {
			https: retryCount === 0 ? proxy : undefined
		}
	});

	stream.retryCount = retryCount;

	stream.on('retry', (retryCount, error) => {
		console.log('Creating new retry:', retryCount);
		fn(retryCount);
	}).on('error', error => {
		console.log('Retry count:', error.request.retryCount);
	}).resume().on('end', () => {
		console.log('`end` event');
	});
};

fn(0);
```

### `h2c`

There is no direct [`h2c`](https://datatracker.ietf.org/doc/html/rfc7540#section-3.1) support.

However, you can provide a `h2session` option in a `beforeRequest` hook. See [an example](examples/h2c.js).

### Electron `net` module is not supported

Got doesn't support the `electron.net` module. It's missing crucial APIs that are available in Node.js.\
While Got used to support `electron.net`, it got very unstable and caused many errors.

However, you can use [IPC communication](https://www.electronjs.org/docs/api/ipc-main#ipcmainhandlechannel-listener) to get the Response object:

```js
// Main process
const got = require('got');

const instance = got.extend({
	// ...
});

ipcMain.handle('got', async (event, ...args) => {
	const {statusCode, headers, body} = await instance(...args);
	return {statusCode, headers, body};
});

// Renderer process
async () => {
	const {statusCode, headers, body} = await ipcRenderer.invoke('got', 'https://httpbin.org/anything');
	// ...
}
```
