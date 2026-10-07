# IPFly PHP SDK

A dependency-free PHP client for the [IPFly](https://ipfly.world) IP geolocation API. One file — it handles caching, retries, rate limiting, and parallel batch lookups for you.

- **Zero third-party dependencies** — uses the `curl` extension when available, falls back to PHP streams (`file_get_contents`) otherwise
- **TTL + LRU caching** — in-process memory, a shared JSON file, or **APCu** (shared across PHP-FPM workers on the same server, no disk I/O)
- **Resilient** — automatic retry with exponential backoff + jitter on network/5xx failures
- **Real parallel batching** — `batchLookup()` uses `curl_multi` for genuine concurrency, not a sequential loop
- **Client-side rate limiting** — a token-bucket limiter for long-running scripts and batch jobs
- **Typed errors** — a single `IPFlyException` with `->getStatus()` and `->getErrorCode()`
- **PHP 7.4+** — typed properties, arrow functions; no PHP 8-only syntax required

---

## Where this SDK belongs

This client is meant to run **server-side** — in a web backend, a CLI script, a queue worker, a cron job. It is not meant to be shipped to a browser or a distributed client app where your API token could be extracted. If you need geolocation results in a frontend app, put this SDK behind your own endpoint and call that endpoint from the client (see the [Laravel/Slim proxy example](#examples) below), or use the [IPFly JavaScript SDK](./javascript-sdk.html) with a domain-restricted token for prototyping.

---

## Installation

No Composer package is published — copy `ipfly_sdk.php` into your project.

```bash
# from your project root
curl -O https://your-host/ipfly_sdk.php
# or just drop the file into your source tree, e.g. src/IPFly/ipfly_sdk.php
```

```php
<?php
require_once __DIR__ . '/ipfly_sdk.php';

use IPFly\IPFlyClient;
use IPFly\IPFlyException;

$client = new IPFlyClient('YOUR_TOKEN');
```

> If you *do* use Composer in your project, autoloading still works fine — just `require_once` this file once (e.g. from a service provider or bootstrap file) since it isn't PSR-4-mapped as its own package.

### Requirements

- PHP 7.4 or later
- The `curl` extension is **optional but recommended** — enables real parallel batch lookups and more precise timeouts. Without it, the SDK automatically falls back to PHP's built-in stream wrappers for single lookups and runs `batchLookup()` sequentially.
- The `apcu` extension is **optional** — only needed if you choose `'storage' => 'apcu'` for the cache.

---

## Quick start

```php
<?php
require_once 'ipfly_sdk.php';

use IPFly\IPFlyClient;
use IPFly\IPFlyException;

$client = new IPFlyClient('YOUR_TOKEN', [
    'include' => 'security', // optional: request the security/ASN field set on every call
]);

// Look up a specific IP
$data = $client->lookup('8.8.8.8');
echo $data['city'], ' ', $data['country_name'], ' ', $data['security']['is_vpn'] ? 'VPN' : 'not VPN', "\n";

// Look up the caller's own IP (omit the ip argument)
$me = $client->lookupSelf();
echo "You appear to be in {$me['city']}\n";

// Handle errors explicitly
try {
    $client->lookup('not-an-ip');
} catch (IPFlyException $err) {
    echo $err->getErrorCode(), ': ', $err->getMessage(), "\n";
}
```

---

## Configuration reference

The second constructor argument is an options array: `new IPFlyClient($token, [...])`.

| Option | Type | Default | Description |
|---|---|---|---|
| `base_url` | `string` | `https://ipfly.world/api` | Override to point at your own backend proxy. |
| `include` | `string` | `null` | Default `include` value sent with every request (e.g. `"security"`). Overridable per call. |
| `timeout` | `float` | `8.0` | Per-request timeout in seconds. |
| `retries` | `int` | `2` | Retry attempts for network errors and `5xx` responses. `4xx` is never retried. |
| `retry_delay` | `float` | `0.3` | Base delay (seconds) for exponential backoff; jitter is added automatically. |
| `rate_limit` | `float` | `0` (unlimited) | Max requests/second this client will issue. |
| `concurrency` | `int` | `6` | Default max parallel cURL handles for `batchLookup()`. |
| `cache` | `array` | see below | Response cache settings (see next table). |
| `debug` | `bool` | `false` | Write internal lifecycle events to `STDERR` (tokens are redacted). |
| `on_request` | `callable` | `null` | `function(?string $ip, string $url): void` — fired right before a request is sent. |
| `on_response` | `callable` | `null` | `function(?string $ip, array $data, bool $fromCache): void` — fired after a successful lookup. |
| `on_error` | `callable` | `null` | `function(?string $ip, IPFlyException $error): void` — fired when a lookup ultimately fails. |
| `on_cache_hit` | `callable` | `null` | `function(?string $ip, array $data): void` — fired when a cached response is served instead of a network call. |

### `cache` array keys

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `bool` | `true` | Set `false` to disable caching entirely. |
| `ttl` | `float` | `300` | How long a cached response stays valid, in **seconds**. |
| `max_size` | `int` | `500` | Max cached entries before least-recently-used entries are evicted. |
| `storage` | `'memory' \| 'file' \| 'apcu'` | `'memory'` | See below. |
| `path` | `string` | system temp dir | File path used when `storage => 'file'`. |
| `prefix` | `string` | `'ipfly:'` | Key prefix used when `storage => 'apcu'`. |

**Which storage should I pick?**
- `'memory'` — cache lives only for the current request/process. Fine for CLI scripts and short jobs; useless for a typical PHP-FPM web app where each request is a fresh process.
- `'file'` — a shared JSON file on disk, persisted across requests and processes. Simple and portable; rewrites the whole file on every write, so it's best at low/medium traffic.
- `'apcu'` — shared in-memory cache across PHP-FPM workers on the same machine, no disk I/O. The best choice for a typical web app, if the `apcu` extension is installed.

```php
$client = new IPFlyClient('YOUR_TOKEN', [
    'cache' => ['storage' => 'apcu', 'ttl' => 3600],
]);
```

---

## API

### `lookup(?string $ip = null, array $options = []): array`
Look up a single IP. Pass `null` (or omit) to geolocate the caller.

```php
$client->lookup('1.1.1.1', [
    'include' => 'security', // overrides the client-level `include` for this call only
    'skip_cache' => true,     // force a fresh network request, bypassing the cache
    'timeout' => 3.0,          // override the client's default timeout for this call
]);
```
Returns the parsed JSON response as an associative array (see [Response shape](#response-shape)). Throws `IPFlyException` on failure.

### `lookupSelf(array $options = []): array`
Shorthand for `lookup(null, $options)` — geolocates the caller's own IP.

### `batchLookup(array $ips, ?int $concurrency = null, array $options = []): BatchResult[]`
Looks up an array of IPs. When the `curl` extension is loaded, this runs genuinely in parallel via `curl_multi`, bounded by `$concurrency` (or the client's default); otherwise it falls back to a sequential loop automatically. This call **never throws** for an individual IP failure — it returns a list of `BatchResult` in the same order as `$ips`:

```php
use IPFly\BatchResult;

$results = $client->batchLookup(['8.8.8.8', '1.1.1.1', 'not-an-ip']);
foreach ($results as $r) {
    if ($r->ok) {
        echo "{$r->ip} -> {$r->data['city']}\n";
    } else {
        echo "{$r->ip} failed: {$r->error->getErrorCode()}\n";
    }
}
```

`BatchResult` is a small value object: `public string $ip`, `public bool $ok`, `public ?array $data`, `public ?IPFlyException $error`.

### `clearCache(): void`
Empties the response cache immediately.

### `cacheStats(): array`
Returns `['enabled' => ..., 'size' => ..., 'ttl' => ..., 'max_size' => ...]` — useful for logging or a health-check endpoint.

### `IPFly\is_valid_ip(string $ip): bool`
Namespaced function — validates an IPv4 or IPv6 string without making a request.

```php
use function IPFly\is_valid_ip;

is_valid_ip('8.8.8.8');               // true
is_valid_ip('2001:4860:4860::8888');  // true
is_valid_ip('not-an-ip');             // false
```

---

## Response shape

Successful lookups return the same JSON structure the IPFly API returns, unchanged, as a PHP associative array — see the [IPFly API docs](https://ipfly.world) for the full field reference. Shape depends on your plan and the `include` value requested (`security` and ASN/company fields require Pro or higher).

```php
[
    'ip' => '8.8.8.8',
    'hostname' => 'dns.google',
    'country_name' => 'United States',
    'city' => 'Mountain View',
    'latitude' => '37.4056',
    'longitude' => '-122.0775',
    'time_zone' => ['name' => 'America/New_York', /* ... */],
    'asn' => ['asn' => 'AS15169', 'name' => 'Google LLC', /* ... */],
    'security' => ['is_vpn' => false, 'is_tor' => false, /* ... */],
]
```

---

## Error handling

All failures throw `IPFly\IPFlyException`, a `RuntimeException` subclass with:

| Method | Description |
|---|---|
| `getMessage()` | Human-readable description. |
| `getStatus()` | HTTP status code, if the request reached the server (`null` for network/timeout errors). |
| `getErrorCode()` | Machine-readable code — see table below. |
| `getCause()` | The underlying error info or decoded response body, when available. |

| Code | Meaning |
|---|---|
| `MISSING_TOKEN` | No token was supplied when creating the client. |
| `INVALID_IP` | The IP string failed local validation before any request was sent. |
| `TIMEOUT` | The request exceeded `timeout`. |
| `NETWORK_ERROR` | The request failed before a response was received (offline, DNS, connection refused, etc). |
| `HTTP_ERROR` / server-provided code | The API responded with a non-2xx status. Check `getStatus()` (401 = bad token, 403 = plan/permission issue, 404 = private/bogon IP, 429/5xx = retry-worthy). |
| `BAD_RESPONSE` | The server responded but the body wasn't valid JSON, or the response was empty. |

```php
use IPFly\IPFlyClient;
use IPFly\IPFlyException;

$client = new IPFlyClient('YOUR_TOKEN');

try {
    $client->lookup('8.8.8.8');
} catch (IPFlyException $err) {
    if ($err->getErrorCode() === 'TIMEOUT') {
        // retry later, log, fall back to a default
    } elseif ($err->getStatus() === 401) {
        // token is invalid — surface a config error, don't retry
    }
    error_log($err->getMessage());
}
```

---

## Performance notes

- **Caching** avoids re-querying the same IP within the TTL window. Use `'apcu'` storage for a typical PHP-FPM web app so the cache is actually shared across requests, not reset every time (unlike `'memory'`, which only lives for one process/request).
- **Parallel batching**: `batchLookup()` uses `curl_multi` so N IPs don't cost N × round-trip-time — they're bounded by `concurrency` concurrent connections instead. Failed requests are retried individually within the same batch, respecting the retry/backoff settings.
- **Rate limiting** is a courtesy limiter on the client side — it smooths bursts (e.g. a bulk-enrichment script) so you don't blow through your plan's requests/second in a tight loop.
- **Retries** only apply to transient failures (network errors, timeouts, `5xx`). A `401`/`403`/`404` fails fast since retrying won't fix a bad token or a private IP.

---

## Examples

### Enriching a CSV of visitor IPs

```php
<?php
require_once 'ipfly_sdk.php';

use IPFly\IPFlyClient;

$client = new IPFlyClient(getenv('IPFLY_TOKEN'), [
    'cache' => ['storage' => 'file', 'path' => __DIR__ . '/geo_cache.json'],
]);

$ips = [];
if (($handle = fopen('visitors.csv', 'r')) !== false) {
    $header = fgetcsv($handle);
    $ipCol = array_search('ip', $header, true);
    while (($row = fgetcsv($handle)) !== false) {
        $ips[] = $row[$ipCol];
    }
    fclose($handle);
}

foreach ($client->batchLookup($ips, 8) as $result) {
    if ($result->ok) {
        echo "{$result->ip} -> {$result->data['country_name']}\n";
    } else {
        echo "{$result->ip} -> error: {$result->error->getErrorCode()}\n";
    }
}
```

### A minimal proxy endpoint (keep the token server-side)

```php
<?php
// public/api/geo.php
require_once __DIR__ . '/../../ipfly_sdk.php';

use IPFly\IPFlyClient;
use IPFly\IPFlyException;

$client = new IPFlyClient(getenv('IPFLY_TOKEN'));
$ip = $_GET['ip'] ?? $_SERVER['REMOTE_ADDR'];

header('Content-Type: application/json');

try {
    echo json_encode($client->lookup($ip, ['include' => 'security']));
} catch (IPFlyException $err) {
    http_response_code($err->getStatus() ?: 502);
    echo json_encode(['error' => $err->getErrorCode(), 'message' => $err->getMessage()]);
}
```

### Laravel controller

```php
<?php

namespace App\Http\Controllers;

use IPFly\IPFlyClient;
use IPFly\IPFlyException;
use Illuminate\Http\Request;

class GeoController extends Controller
{
    public function __invoke(Request $request)
    {
        static $client = null;
        $client ??= new IPFlyClient(config('services.ipfly.token'), [
            'cache' => ['storage' => 'apcu'],
        ]);

        try {
            return response()->json(
                $client->lookup($request->query('ip') ?? $request->ip(), ['include' => 'security'])
            );
        } catch (IPFlyException $err) {
            return response()->json(
                ['error' => $err->getErrorCode(), 'message' => $err->getMessage()],
                $err->getStatus() ?: 502
            );
        }
    }
}
```

Your frontend (browser JS, mobile app, etc.) calls your own `/api/geo` route — the IPFly token stays in this process and is never sent to the client.

---

## License

MIT — use it, fork it, ship it.
