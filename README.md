# SEON Web Stream SDK

The SEON Web Stream SDK continuously collects behavioral signals from your web application — click patterns, typing dynamics, clipboard activity, navigation and more — and streams them to the SEON platform for real time stream analysis and fraud detection. The SDK is distributed as an ESM package via NPM, and it works with React applications and Vanilla JavaScript. Although other web frameworks could work as well, they will need to be thoroughly tested to ensure compatibility. Single-page applications (SPAs) work best generally.

## Requirements

- A build system with ES2023 support (for external dependency resolution and optional polyfills). Example setup: Vite 6+ with SWC 1.8+ and core-js 3.25+.
- A browser with native [BigInt](https://caniuse.com/bigint) support (Chrome 67+, Firefox 68+, Safari 14+, Edge 79+).
- `localStorage` enabled (used for stream persistence).

## Installation

```bash
npm install @seontechnologies/seon-stream-sdk-web
# or
bun add @seontechnologies/seon-stream-sdk-web
```

Other package managers that support npm registries will also work. Modern ones (npm 7+, bun, pnpm 8+) automatically install the SDK's runtime peer dependencies. We declare identity sensitive packages (eg. `rxjs`, `zod`) as peer dependencies so they share a single instance between your app and the SDK, avoiding bugs from having duplicate copies in the dependency tree.

Dependency version ranges are intentionally pinned to specific minor versions (e.g. `~7.8.2`). We bump these versions in regular SDK releases as newer versions get validated by our tests. If this conflicts with another library in your project, you can manually override the resolved version via your package manager's `overrides` (npm/bun) or `resolutions` (yarn) field:

```json
{
  "overrides": {
    "rxjs": "7.7.0"
  }
}
```

Then import the SDK in your application:

```js
import SeonStream from "@seontechnologies/seon-stream-sdk-web";
```

## Getting started

### Configuration

The SDK accepts an optional configuration object at construction time. All options have sensible defaults except `authData`, which must be set before starting a stream.

| Option                        | Type      | Default                                      | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ----------------------------- | --------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `authData`                    | `string`  | `undefined`                                  | The Base64 auth blob your backend obtains from SEON's auth endpoint. The SDK decodes it into a JWT token, ingest domains and additional configuration. Must be set via the constructor or `setConfig()` before calling `startStream()`. See [Authentication](#authentication).                                                                                                                                                                                |
| `sessionTimeoutMs`            | `number`  | `0`                                          | The maximum time (in ms) between the last recorded event and a new `startStream()` call for the SDK to continue the previous stream with the same stream ID. When a stream is continued, the existing event history is preserved and new events are appended to it. If the elapsed time exceeds this value, or if the label has changed, a new stream ID is generated. Set to `0` to always start a fresh stream. Maximum value is 48 hours (172,800,000 ms). |
| `enableLeaveDialog`           | `boolean` | `false`                                      | When `true`, the browser displays a confirmation dialog when the user navigates away or attempts to refresh/close the tab, ensuring the SDK has enough time to send the latest interaction data. Not recommended in non-SPA apps with navigation.                                                                                                                                                                                                             |
| `silentMode`                  | `boolean` | `true`                                       | When `false`, the SDK enables open port scanning, which may generate several error logs in the console.                                                                                                                                                                                                                                                                                                                                                       |
| `elementTagKey`               | `string`  | `'x-stream-tag'`                             | The HTML attribute name used for declarative element tagging (see [Tagging](#tagging)).                                                                                                                                                                                                                                                                                                                                                                       |
| `apiEndpoint`                 | `string`  | `undefined`                                  | Optional override of the ingest URL. When set, the SDK posts events there instead of selecting from the ingest domains in `authData`. Useful for routing through your own first-party reverse proxy. See [Proxying the ingest traffic](#proxying-the-ingest-traffic-optional).                                                                                                                                                                                |
| `storageKey`                  | `string`  | `'x_stream'`                                 | The `localStorage` key prefix used for stream persistence.                                                                                                                                                                                                                                                                                                                                                                                                    |
| `eventBufferKeyPostfix`       | `string`  | `'eb'`                                       | Postfix appended to `storageKey` to form the `localStorage` key for the user events (e.g. `'x_stream_eb'`).                                                                                                                                                                                                                                                                                                                                                   |
| `aggregationBufferKeyPostfix` | `string`  | `'ab'`                                       | Postfix appended to `storageKey` to form the `localStorage` key for the user aggregated data (e.g. `'x_stream_ab'`).                                                                                                                                                                                                                                                                                                                                          |
| `resolverDomain`              | `string`  | `'seonintelligenceresolver.com'`             | The domain used for network intelligence resolver requests.                                                                                                                                                                                                                                                                                                                                                                                                   |
| `resolverEndpointTl`          | `string`  | `'eb6a7d55b667d9b6e52e2ebe363274d7b395eb78'` | The subdomain identifier for the resolver endpoint. Combined with `resolverDomain` they form the full URL.                                                                                                                                                                                                                                                                                                                                                    |

Configuration values are validated on input. Any properties that fail validation are not applied and are returned to the caller (see [Updating configuration](#updating-configuration)).

### Authentication

The SDK authenticates with a short lived, server issued token that you need to get separately:

1. Your backend requests auth data from SEON's auth endpoint (server to server with your own SEON API key).
2. Your app passes that blob to the SDK via the `authData` config property before starting a stream.
3. The SDK decodes it into a JWT and the ingest domains.

Since the token is short lived, fetch a fresh one before every stream start using the endpoint for your account's region:

| Region           | Endpoint                                                            |
| ---------------- | ------------------------------------------------------------------- |
| EU (Ireland)     | `https://api.seon.io/session-monitoring-api/v1/auth`                |
| US (N. Virginia) | `https://api.us-east-1-main.seon.io/session-monitoring-api/v1/auth` |

Example flow:

```bash
# 1. Your backend serves /auth for the client to call SEON's auth API.
curl -X POST https://api.seon.io/session-monitoring-api/v1/auth \
  -H "X-API-KEY: YOUR_SEON_API_KEY"
```

```js
// 2. Your client fetches auth data from your backend.
const authData = await fetch("/auth", { method: "POST" }).then((res) =>
  res.text(),
);

// 3. Pass it to the SDK via the constructor…
const seonStream = new SeonStream({ authData });
// …or apply it to an existing instance.
seonStream.setConfig({ authData });
```

#### Proxying the ingest traffic (optional)

Set the `apiEndpoint` config property to route ingest through your own first party reverse proxy. The SDK then posts there instead of the domains from `authData`. Forward all requests and responses as is, and especially allow the `Authorization`, `Content-Type`, and `X-Client-Type` request headers and expose the `X-New-Token` response header for token rotation to work. Network intelligence requests cannot be proxied as they need to directly originate from the client.

To get the upstream domains from the SDK, read `remoteConfig.domains` returned by `setConfig()` or the buffered `constructor` debug message. Using the first one is fine and expected but you could use the other domains as well. Declare the network protocol and append the appropriate route to construct the proper endpoint.

```js
const { remoteConfig } = seonStream.setConfig({});
// OR
seonStream.onDebug = (dbg) => {
  if (dbg.message === "constructor") {
    const { remoteConfig } = dbg.data.config;
  }
};

const { domains } = remoteConfig;
const domain = domains[0];
const endpoint = `https://${domain}/api/v1/events`;
```

### Initialization

Create a single SDK instance — the constructor enforces the singleton pattern. Typically, initialize the SDK as early as possible in your application lifecycle:

```js
const seonStream = new SeonStream({
  authData: authDataFromYourBackend,
  sessionTimeoutMs: 60_000,
  enableLeaveDialog: true,
  silentMode: false,
});
```

Invalid configuration properties passed to the constructor are ignored and returned via the `onError` callback.

### Updating configuration

You can update configuration after initialization using `setConfig`. Properties that fail validation are returned in the `rejected` object and are not applied:

```js
const { config, rejected } = seonStream.setConfig({
  sessionTimeoutMs: 600_000,
});

if (Object.keys(rejected).length > 0) {
  /**
   * Inspect invalid config fields in `rejected`.
   */
}
```

## Starting and stopping a stream

### Starting a stream

Call `startStream()` with a `Metadata` object to begin collecting and streaming behavioral data. The `Metadata` object accepts the following properties:

| Property | Type     | Constraint                       | Description                                                                                                                                                                                                                               |
| -------- | -------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `label`  | `string` | Optional. Maximum 32 characters. | A label identifying the stream (e.g. `'checkout-flow'`, `'login'`). Used by the SDK to determine whether a stream can be continued across page reloads — a new stream ID is generated if the label changes between `startStream()` calls. |

```js
try {
  const result = await seonStream.startStream({ label: "checkout-flow" });
  const streamId = result.streamId;
} catch (err) {
  /**
   * Failed to start the stream, see `err` for more details.
   */
}
```

The returned object contains:

| Property                | Type                      | Description                                                                                                 |
| ----------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `streamId`              | `string`                  | The unique identifier for this stream. Send this to the Fraud API to correlate with SEON's risk assessment. |
| `metadata`              | `Readonly<Metadata>`      | A read-only copy of the stream metadata.                                                                    |
| `metadataRejectedProps` | `Record<string, unknown>` | Any metadata properties that failed validation (e.g. `label` exceeding 32 characters).                      |

> **Note:** `startStream()` will throw if a stream is already running. Always call `finishStream()` before starting a new stream.

### Stopping a stream

```js
try {
  await seonStream.finishStream();
} catch (err) {
  /**
   * Failed to finish the stream, see `err` for more details.
   */
}
```

> **Note:** `finishStream()` will throw if no stream is currently running.

### Stream continuation

The SDK can continue a previous stream across page reloads and navigations, preserving the same stream ID and event history. A stream is continued when all of the following conditions are met:

- `sessionTimeoutMs` is set to a positive value.
- The time since the last recorded event has not exceeded `sessionTimeoutMs`.
- The `label` passed to `startStream()` matches the label of the previous stream.

If any condition is not met, a new stream ID is generated and the stream starts fresh.

## Custom events

Send named custom events with optional additional data during an active stream:

```js
seonStream.createCustomEvent("purchase_attempt", '{"amount":99.99}');
```

| Parameter | Type     | Constraint                          |
| --------- | -------- | ----------------------------------- |
| `name`    | `string` | Required. Maximum 32 characters.    |
| `data`    | `string` | Optional. Maximum 1,024 characters. |

If the constraints are violated, then the `onError` callback gets invoked and the event is not recorded. The stream must be running — calling `createCustomEvent` without an active stream will throw.

## Handling SDK errors

When the SDK encounters an error, it invokes the `onError` callback and continues. This callback can be used for in-house metrics, or to identify potential issues with the integration.

```ts
seonStream.onError = (err: Error) => {
  /**
   * A client-side error occurred.
   * See `err` for more details.
   */
};
```

## Handling SDK debug messages

The SDK emits optional debug messages, which can be captured via the `onDebug` callback. This can be useful for validation or to identify misconfiguration.

```ts
seonStream.onDebug = (dbg: Debug) => {
  /* The debug message is contained within `dbg`. */
};
```

The `Debug` interface:

```ts
interface Debug {
  scope: string;
  message: string;
  timestamp: string; // ISO format
  data?: Record<string, any>;
}
```

## Handling stream timeouts

The SEON backend will terminate a stream when it reaches the maximum stream duration (48 hours). To handle this, set the `onTimeout` callback:

```js
seonStream.onTimeout = () => {
  /**
   * Stream was terminated by the server.
   * We should start a new stream here.
   */
};
```

When a server-side timeout occurs, the SDK automatically calls `finishStream()` before invoking the callback. You do not need to stop the stream manually.

## Handling authentication errors

If the server rejects a request with HTTP 401 (invalid or expired session token), the SDK automatically calls `finishStream()` before invoking the `onAuthError` callback. This typically means the `authData` was missing, malformed, or its session token has expired. Obtain a fresh `authData` blob from your backend, apply it with `setConfig({ authData })`, and start a new stream.

```js
seonStream.onAuthError = () => {
  /* Invalid or expired session token. Re-authenticate with a fresh authData. */
};
```

## Handling IP bans

If the server rejects a request with HTTP 403 (IP banned), the `onIpBan` callback is invoked. The stream is not stopped — the SDK automatically retries with a longer backoff interval. You can use the callback to e.g. log the event:

```js
seonStream.onIpBan = () => {
  /**
   * The IP address is temporarily banned.
   */
};
```

## Tagging

Tag HTML elements with human-readable names so tracked interactions are easier to interpret on the SEON dashboard.

### HTML attribute tagging

Add the `x-stream-tag` attribute (or your custom `elementTagKey`) directly in your HTML:

```html
<form x-stream-tag="login-form">
  <input x-stream-tag="email-field" name="email" type="email" />
  <input x-stream-tag="password-field" name="password" type="password" />
  <button x-stream-tag="submit-btn" type="submit">Log in</button>
</form>
```

### JavaScript tagging

Use `setElementTags()` to assign tags from JavaScript. This is useful for dynamically generated elements or frameworks where you cannot easily add HTML attributes:

```js
const emailInput = document.getElementById("email-input");
const passwordInput = document.getElementById("password-input");

seonStream.setElementTags(
  new Map([
    [emailInput, "email-field"],
    [passwordInput, "password-field"],
  ]),
);
```

Tags set via `setElementTags()` take the highest priority in the element identification chain.

### Element identification priority

When the SDK records an interaction, it identifies the target element using the following resolution chain (first non-empty value wins):

| Priority | Source                           | Example                                      |
| -------- | -------------------------------- | -------------------------------------------- |
| 1        | `setElementTags()` map           | `'email-field'`                              |
| 2        | HTML attribute (`elementTagKey`) | `x-stream-tag="email-field"`                 |
| 3        | `name` attribute                 | `<input name="email">` → `'email'`           |
| 4        | `id` attribute                   | `<input id="email-input">` → `'email-input'` |
| 5        | Tag name (`nodeName`)            | `<input>` → `'INPUT'`                        |

## Automatic tracking

During an active stream, the SDK automatically captures behavioral signals without additional integration code.

> **Privacy:** The SDK does not capture the actual content of input fields. Only interaction metadata (e.g. focus changes, typing speed, paste actions) is collected, ensuring that sensitive information such as personal data or passwords are never recorded.

### Click tracking

The SDK captures pointer click events, including the target element, button type, and coordinates.

### Input and typing tracking

Text input interactions on `<input>` and `<textarea>` elements are monitored, including:

- Focus and blur transitions
- Typing characteristics (speed, duration, etc.)
- Clipboard activity (copy, paste, cut)
- Browser autofill detection

### Navigation tracking

The SDK tracks page navigation events, including:

- The initial page URL and title when the stream starts
- `history.pushState()` and `history.replaceState()` calls (e.g. in SPA routing)
- Browser back/forward navigation (`popstate` events)

### Visibility and focus tracking

The SDK detects when the page goes to the background or foreground.

### Device and browser intelligence

The SDK collects device and browser signals and streams them to the SEON platform, where they are used to build real-time risk flags — for example, detecting proxy/VPN usage, browser spoofing, automation tools, or private browsing.

## Content-Security-Policy (CSP)

If your website uses Content Security Policy (CSP) headers, ensure that the following sources are allowed:

- `connect-src *.usersession.io *.seonintelligenceresolver.com`

> Note: In case the `*.usersession.io` domain is rotated, please update the CSP accordingly. Alternatively, proxy the ingest traffic as described in [Proxying the ingest traffic](#proxying-the-ingest-traffic-optional).

## Common integration difficulties

- **`authData` must be set before starting a stream.** If no `authData` has been provided before calling `startStream()`, the SDK has no session token and requests will be rejected by the SEON backend. Because `authData` is short-lived, fetch a fresh blob before every stream start.
- **Late initialization.** The SDK begins tracking events only after `startStream()` is called. If you delay the stream start, early user interactions will not be captured. Initialize and start the stream as early as possible in your application lifecycle.
- **Default `sessionTimeoutMs` is `0`.** With the default value, every `startStream()` call creates a new stream ID. If you want streams to survive page reloads or navigations (e.g. in a multi-page checkout flow), set `sessionTimeoutMs` to a positive value and pass the same `label` to each `startStream()` call.
- **`enableLeaveDialog` is `false` by default.** Without this option, the last seconds of interaction data could be lost when the user abruptly closes the tab/browser or navigates away before the SDK's next transmission. When enabled, the browser displays a confirmation dialog on navigation/close, giving the SDK time to send the latest interaction data.
- **Tag elements early.** Tags are resolved at the time events fire. If you tag an element after the user has already interacted with it, those early events will be recorded with the auto-resolved name (e.g. from the `name` or `id` attribute) instead of your custom tag.
- **`localStorage` must be available.** The SDK uses `localStorage` for stream persistence. If `localStorage` is blocked (e.g. by browser privacy settings or iframe sandboxing), stream restoration across page reloads will not work. The SDK bounds its persisted data to fit within the browser's `localStorage` quota (typically 5–10 MB per origin).
- **Data buffering during network outages.** If the network is unavailable, the SDK buffers events in memory (up to 10,000 events, oldest evicted first) and retries transmission automatically. Buffered data is persisted to `localStorage` so it survives page reloads.

## Limitations and auto-tagging behaviour

### Form detection

When an interaction occurs on an element inside a `<form>`, the SDK records the parent form's identity using the same resolution chain as element names. For anonymous forms (no tag, `name`, or `id`), the form name resolves to `'FORM'`.

### Input tracking

| Scenario                                                   | Tracked? | Notes                                                          |
| ---------------------------------------------------------- | -------- | -------------------------------------------------------------- |
| `<input>` elements                                         | Yes      | Focus, typing, clipboard, and autofill are captured.           |
| `<textarea>` elements                                      | Yes      | Same as `<input>`.                                             |
| Custom input components (e.g. `contenteditable`, `canvas`) | No       | Only native `<input>` and `<textarea>` elements are supported. |

### Click tracking

| Scenario                             | Tracked? | Notes                                                                                                                                                      |
| ------------------------------------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Any `HTMLElement`                    | Yes      | All pointer click events on HTML elements are captured.                                                                                                    |
| Non-HTML targets (e.g. `SVGElement`) | No       | Only `HTMLElement` instances are recorded. SVG elements wrapped in an `HTMLElement` (e.g. a `<button>` containing an `<svg>`) are captured via the parent. |

### Navigation tracking

| Scenario                                 | Tracked? | Notes                                                                                                                  |
| ---------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------- |
| Initial page load                        | Yes      | Emitted automatically when the stream starts.                                                                          |
| `history.pushState()` / `replaceState()` | Yes      | Detected via History API patching.                                                                                     |
| Browser back / forward                   | Yes      | Detected via `popstate` event.                                                                                         |
| Full page reload / navigation            | Partial  | Data is persisted to `localStorage`; the new page emits a fresh navigation event if the stream is continued.           |
| Hash changes (`hashchange`)              | Partial  | Detected only if triggered via `popstate`. Direct `location.hash` assignments without `pushState` may not be captured. |
