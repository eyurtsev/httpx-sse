# httpx-sse

[![Build Status](https://dev.azure.com/florimondmanca/public/_apis/build/status/florimondmanca.httpx-sse?branchName=master)](https://dev.azure.com/florimondmanca/public/_build?definitionId=19)
[![Coverage](https://codecov.io/gh/florimondmanca/httpx-sse/branch/master/graph/badge.svg)](https://codecov.io/gh/florimondmanca/httpx-sse)
[![Package version](https://badge.fury.io/py/httpx-sse.svg)](https://pypi.org/project/httpx-sse)

Consume [Server-Sent Event (SSE)](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events) messages with [HTTPX](https://www.python-httpx.org).

**Table of contents**

- [Installation](#installation)
- [Quickstart](#quickstart)
- [How-To](#how-to)
- [API Reference](#api-reference)

## Installation

**NOTE**: This is alpha software. Please be sure to pin your dependencies.

```bash
# --Unreleased--
pip install git+https://github.com/florimondmanca/httpx-sse.git
```

## Quickstart

`httpx-sse` provides the [`connect_sse`](#connect_sse) and [`aconnect_sse`](#aconnect_sse) helpers for connecting to an SSE endpoint. The resulting [`EventSource`](#eventsource) object exposes the [`.iter_sse()`](#iter_sse) and [`.aiter_sse()`](#aiter_sse) methods to iterate over the server-sent events.

Example usage:

```python
import httpx
from httpx_sse import connect_sse

with httpx.Client() as client:
    with connect_sse(client, "http://localhost:8000/sse") as event_source:
        for sse in event_source.iter_sse():
            print(sse.event, sse.data, sse.id, sse.retry)
```

You can try this against this example Starlette server ([credit](https://sysid.github.io/sse/)):

```python
# Requirements: pip install uvicorn starlette sse-starlette
import asyncio
import uvicorn
from starlette.applications import Starlette
from starlette.routing import Route
from sse_starlette.sse import EventSourceResponse

async def numbers(minimum, maximum):
    for i in range(minimum, maximum + 1):
        await asyncio.sleep(0.9)
        yield {"data": i}

async def sse(request):
    generator = numbers(1, 5)
    return EventSourceResponse(generator)

routes = [
    Route("/sse", endpoint=sse)
]

app = Starlette(routes=routes)

if __name__ == "__main__":
    uvicorn.run(app)
```

## How-To

### Calling into Python web apps

You can [call into Python web apps](https://www.python-httpx.org/async/#calling-into-python-web-apps) with HTTPX and `httpx-sse` to test SSE endpoints directly.

Here's an example of calling into a Starlette ASGI app...

```python
import asyncio

import httpx
from httpx_sse import aconnect_sse
from sse_starlette.sse import EventSourceResponse
from starlette.applications import Starlette
from starlette.routing import Route

async def auth_events(request):
    async def events():
        yield {
            "event": "login",
            "data": '{"user_id": "4135"}',
        }

    return EventSourceResponse(events())

app = Starlette(routes=[Route("/sse/auth/", endpoint=auth_events)])

async def main():
    async with httpx.AsyncClient(app=app) as client:
        async with aconnect_sse(
            client, "http://localhost:8000/sse/auth/"
        ) as event_source:
            events = [sse async for sse in event_source.aiter_sse()]
            (sse,) = events
            assert sse.event == "login"
            assert sse.json() == {"user_id": "4135"}

asyncio.run(main())
```

### Handling reconnections

_(Advanced)_

`SSETransport` and `AsyncSSETransport` don't have reconnection built-in. This is because how to perform retries is generally dependent on your use case. As a result, if the connection breaks while attempting to read from the server, you will get an `httpx.ReadError` from `iter_sse()` (or `aiter_sse()`).

However, `httpx-sse` does allow implementing reconnection by using the `Last-Event-ID` and reconnection time (in milliseconds), exposed as `sse.id` and `sse.retry` respectively.

Here's how you might achieve this using [`stamina`](https://github.com/hynek/stamina)...

```python
import time
from typing import Iterator

import httpx
from httpx_sse import iter_sse, ServerSentEvent
from stamina import retry

def iter_sse_retrying(client, url):
    last_event_id = ""
    reconnection_delay = 0.0

    # `stamina` will apply jitter and exponential backoff on top of
    # the `retry` reconnection delay sent by the server.
    @retry(on=httpx.ReadError)
    def _iter_sse():
        nonlocal last_event_id, reconnection_delay

        time.sleep(reconnection_delay)

        headers = {"Accept": "text/event-stream"}

        if last_event_id:
            headers["Last-Event-ID"] = last_event_id

        with connect_sse(client, url, headers=headers) as event_source:
            for sse in event_source.iter_sse():
                last_event_id = sse.id

                if sse.retry is not None:
                    reconnection_delay = sse.retry / 1000

                yield sse

    return _iter_sse()
```

Usage:

```python
with httpx.Client() as client:
    for iter_sse_retrying(client, "http://localhost:8000/sse") as sse:
        print(sse.event, sse.data)
```

## API Reference

### `connect_sse`

```python
def connect_sse(
    client: httpx.Client,
    url: Union[str, httpx.URL],
    **kwargs,
) -> ContextManager[EventSource]
```

Connect to an SSE endpoint and return an [`EventSource`](#eventsource) context manager.

This sets `Cache-Control: no-store` on the request, as per the SSE spec, as well as `Accept: text/event-stream`.

If the response `Content-Type` is not `text/event-stream`, this will raise an [`SSEError`](#sseerror).

### `aconnect_sse`

```python
async def aconnect_sse(
    client: httpx.AsyncClient,
    url: Union[str, httpx.URL],
    **kwargs,
) -> AsyncContextManager[EventSource]
```

An async equivalent to [`connect_sse`](#connect_sse).

### `EventSource`

```python
def __init__(response: httpx.Response)
```

Helper for working with an SSE response.

#### `response`

The underlying [`httpx.Response`](https://www.python-httpx.org/api/#response).

#### `iter_sse`

```python
def iter_sse() -> Iterator[ServerSentEvent]
```

Decode the response content and yield corresponding [`ServerSentEvent`](#serversentevent).

Example usage:

```python
for sse in event_source.iter_sse():
    ...
```

#### `aiter_sse`

```python
async def iter_sse() -> AsyncIterator[ServerSentEvent]
```

An async equivalent to `iter_sse`.

### `ServerSentEvent`

Represents a server-sent event.

* `event: str` - Defaults to `"message"`.
* `data: str` - Defaults to `""`.
* `id: str` - Defaults to `""`.
* `retry: str | None` - Defaults to `None`.

Methods:

* `json() -> Any` - Returns `sse.data` decoded as JSON.

### `SSEError`

An error that occurred while making a request to an SSE endpoint.

Parents:

* `httpx.TransportError`

## License

MIT
