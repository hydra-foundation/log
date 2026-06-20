# Hydra Log

A minimal PSR-3 logger that writes one plain-text line per record to a writable
stream. No handlers, no processors, no formatters config — Hydra's logging is a
single deliberate class, the data-layer-style "ship the verb" of logging.

## How it works

`StreamLogger` is constructed with an already-open, writable stream — a file
handle, `php://stderr`, `php://memory` in tests. **Sink selection is the
caller's concern**: the class only formats and writes. The app's service
provider is what opens `LOG_PATH` (falling back to stderr when the path is
unwritable) and hands the stream in.

```php
use Hydra\Log\StreamLogger;

$logger = new StreamLogger(fopen('php://stderr', 'w'));
$logger->info('request handled', ['method' => 'GET', 'status' => 200]);
// [2026-06-20T01:44:06+00:00] INFO: request handled {"method":"GET","status":200}
```

Each record renders as `[ISO-8601] LEVEL: message <context>`:

- **Placeholders** — `{key}` tokens are interpolated from context per PSR-3.
  Substitution is keyed on value *type*, not truthiness, so `0`, `''` and
  `false` render rather than vanish. A placeholder with no usable value is left
  intact.
- **Leftover context** is appended as JSON. A `Throwable` under the conventional
  `exception` key is rendered as a readable class/message/file/line + trace
  instead.

## Failures never escalate

A broken sink (closed stream, full disk) must not take the application down with
it: the write is `@fwrite` and guarded by `is_resource`. Logging is best-effort
by design.

## What it deliberately is not

No log rotation, no level filtering, no multiple sinks. Those are the app's job
if it ever needs them — bind a different `Psr\Log\LoggerInterface` implementation
(Monolog, a fan-out logger) in the service provider and every consumer follows,
because consumers depend on the PSR-3 interface, never on this class.
