[> Back to homepage](../readme.md#documentation)

## Hooks API

### `hooks`

**Type: `object<string, Function[]>`**

This option represents the hooks to run.

#### `init`

**Type: `InitHook[]`**\
**Default: `[]`**

```ts
(plainRequestOptions: OptionsInit, options: Options) => void
```

Called with plain request options, right before their normalization.\
The second argument represents the current [`Options`](2-options.md) instance.

**Note:**
> - This is called every time options get merged.

**Note:**
> - This hook is called when a new instance of `Options` is created.
> - Do not confuse this with the creation of `Request` or `got(…)`.

**Note:**
> - When using `got(url, undefined, defaults)` this hook will **not** be called.

This is especially useful in conjunction with `got.extend()` when the input needs custom handling.

For example this can be used to fix typos to migrate from older versions faster.

```js
import got from 'got';

const instance = got.extend({
	hooks: {
		init: [
			plain => {
				if ('followRedirects' in plain) {
					plain.followRedirect = plain.followRedirects;
					delete plain.followRedirects;
				}
			}
		]
	}
});

// Normally, the following would throw:
const response = await instance(
	'https://example.com',
	{
		followRedirects: true
	}
);

// There is no option named `followRedirects`, but we correct it in an `init` hook.
```

Or you can create your own option and store it in a context:

```js
import got from 'got';

const instance = got.extend({
	hooks: {
		init: [
			(plain, options) => {
				if ('secret' in plain) {
					options.context.secret = plain.secret;
					delete plain.secret;
				}
			}
		],
		beforeRequest: [
			options => {
				options.headers.secret = options.context.secret;
			}
		]
	}
});

const {headers} = await instance(
	'https://httpbin.org/anything',
	{
		secret: 'passphrase'
	}
).json();

console.log(headers.Secret);
//=> 'passphrase'
```

#### `beforeRequest`

**Type: `BeforeRequestHook[]`**\
**Default: `[]`**

```ts
(options: Options) => Promisable<void | Response | ResponseLike>
```

Called right before making the request with `options.createNativeRequestOptions()`.\
This hook is especially useful in conjunction with `got.extend()` when you want to sign your request.

**Note:**
> - Got will make no further changes to the request before it is sent.

**Note:**
> - Changing `options.json` or `options.form` has no effect on the request, you should change `options.body` instead. If needed, update the `options.headers` accordingly.

```js
import got from 'got';

const resposne = await got.post(
	'https://httpbin.org/anything',
	{
		json: {payload: 'old'},
		hooks: {
			beforeRequest: [
				options => {
					options.body = JSON.stringify({payload: 'new'});
					options.headers['content-length'] = options.body.length.toString();
				}
			]
		}
	}
);
```

**Tip:**
> - You can indirectly override the `request` function by early returning a [`ClientRequest`-like](https://nodejs.org/api/http.html#http_class_http_clientrequest) instance or a [`IncomingMessage`-like](https://nodejs.org/api/http.html#http_class_http_incomingmessage) instance. This is very useful when creating a custom cache mechanism.
> - [Read more about this tip](https://github.com/sindresorhus/got/blob/docs-v12/documentation/cache.md#advanced-caching-mechanisms).

#### `beforeRedirect`

**Type: `BeforeRedirectHook[]`**\
**Default: `[]`**

```ts
(updatedOptions: Options, plainResponse: PlainResponse) => Promisable<void>
```

The equivalent of `beforeRequest` but when redirecting.

**Tip:**
> - This is especially useful when you want to avoid dead sites.

```js
import got from 'got';

const response = await got('https://example.com', {
	hooks: {
		beforeRedirect: [
			(options, response) => {
				if (options.hostname === 'deadSite') {
					options.hostname = 'fallbackSite';
				}
			}
		]
	}
});
```

### `beforeRetry`

**Type: `BeforeRetryHook[]`**\
**Default: `[]`**

```ts
(error: RequestError) => Promisable<void>
```

**Note:**
> - When using Stream API, this hook is ignored.

**Note:**
> - When retrying, `beforeRequest` hook is called afterwards.

**Note:**
> - If no retry occurs, `beforeError` is called instead.

This hook is especially useful when you want to retrieve the cause of retry.

```js
import got from 'got';

await got('https://httpbin.org/status/500', {
	hooks: {
		beforeRetry: [
			error => {
				console.log(`Retrying: ${error.code}`);
				// Retrying: ERR_NON_2XX_3XX_RESPONSE
			}
		]
	}
});
```

### `afterResponse`

**Type: `AfterResponseHook[]`**\
**Default: `[]`**

```ts
(response: Response, retryWithMergedOptions: (options: OptionsInit) => never) => Promisable<Response | CancelableRequest<Response>>
```

**Note:**
> - When using Stream API, this hook is ignored.

**Note:**
> - Calling the retry function will trigger `beforeRetry` hooks.

Each function should return the response. This is especially useful when you want to refresh an access token.

```js
import got from 'got';

const instance = got.extend({
	hooks: {
		afterResponse: [
			(response, retryWithMergedOptions) => {
				// Unauthorized
				if (response.statusCode === 401) {
					// Refresh the access token
					const updatedOptions = {
						headers: {
							token: getNewToken()
						}
					};

					// Update the defaults
					instance.defaults.options.merge(updatedOptions);

					// Make a new retry
					return retryWithMergedOptions(updatedOptions);
				}

				// No changes otherwise
				return response;
			}
		],
		beforeRetry: [
			error => {
				// This will be called on `retryWithMergedOptions(...)`
			}
		]
	},
	mutableDefaults: true
});
```

### `beforeError`

**Type: `BeforeErrorHook[]`**\
**Default: `[]`**

```ts
(error: RequestError) => Promisable<RequestError>
```

Called with a [`RequestError`](8-errors.md#requesterror) instance. The error is passed to the hook right before it's thrown.

This is especially useful when you want to have more detailed errors.

```js
import got from 'got';

await got('https://api.github.com/repos/sindresorhus/got/commits', {
	responseType: 'json',
	hooks: {
		beforeError: [
			error => {
				const {response} = error;
				if (response && response.body) {
					error.name = 'GitHubError';
					error.message = `${response.body.message} (${response.statusCode})`;
				}

				return error;
			}
		]
	}
});
```
