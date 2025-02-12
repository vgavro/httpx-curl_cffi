# httpx-curl-cffi

[httpx](https://github.com/encode/httpx) transport for
[curl_cffi](https://github.com/lexiforest/curl_cffi) -
python binding for [curl-impersonate fork](https://github.com/lexiforest/curl-impersonate)

Unlike other pure python http clients
like `httpx` (with **native** transport) or `requests`,
`curl_cffi` can impersonate browser's TLS/JA3 and HTTP/2 fingerprints.

Browser simulation implemented by low-level customizations
and usage of native browser TLS libraries
(`BoringSSL` for Chrome, `nss` for Firefox) -
which is impossible to achieve with Python's `OpenSSL` binding.

If you are blocked by some website for no obvious reason,
you can give `curl_cffi` a try.

## Install

```shell
pip install httpx-curl-cffi
```

## Usage

```python
from httpx import Client, AsyncClient
from httpx_curl_cffi import CurlTransport, AsyncCurlTransport, CurlOpt

client = Client(transport=CurlTransport(impersonate="chrome", default_headers=True))
client.get("https://tools.scrapfly.io/api/fp/ja3")

async_client = AsyncClient(transport=AsyncCurlTransport(
  impersonate="chrome",
  default_headers=True,
  # required for parallel requests, see curl_cffi issues below
  curl_options={CurlOpt.FRESH_CONNECT: True}
))
```

## Proxy from environment variables

Note that `httpx.Client` and `httpx.AsyncClient`
disables proxy configuration from environment variables
on providing `transport` argument even with `trust_env=True` (default),
to have this configured in the same way as native transport use snippet below:

```python
import httpx
from httpx._utils import get_environment_proxies
from httpx_curl_cffi import CurlTransport  # or AsyncCurlTransport

def transport_factory(proxy: httpx.Proxy | None = None) -> httpx.BaseTransport:
    return CurlTransport(  # or AsyncCurlTransport
      proxy=proxy,
      verify=False,  # and other custom options
    )

client = httpx.Client(  # or httpx.AsyncClient
    transport=transport_factory(),
    mounts={
        k: transport_factory(httpx.Proxy(url=v)) if v else None
        for k, v in get_environment_proxies().items()
    }
)
```

## TODO

* `httpx.Request` content completely read in memory before sending,
  not sure if it's fixable with `curl_cffi` at all
* `CurlTransport.cert` argument should support in-memory data instead of filenames,
  `pathlib.Path` (instead of strings in `httpx._types.CertTypes`) is forced

## `curl_cffi` issues

* `httpx.Timeout.pool` is ignored, should be implemented in `curl_cffi`
* Simultaneous asynchronous requests requires to set
  `CurlTransport.curl_options={CurlOpt.FRESH_CONNECT: True}`
  <https://github.com/lexiforest/curl_cffi/issues/302>
  <https://github.com/lexiforest/curl_cffi/issues/319>

## Known limitations

* `httpx.Timeout.write` is ignored (`libcurl` limitation)
* `CurlTransport.verify` as `ssl.SSLContext` isn't supported
  (because `OpenSSL` is not used)
* `CurlTransport.trust_env` argument is ignored,
  `libcurl` is always using environment variables for configuration,
  which is disabled for proxies using `CurlOpt.NOPROXY` setting
  to make `proxy` argument have complete control on proxy usage,
  but may have effect in TLS configuration
  (but may not be used by `curl-impersonate` fork, idk)
  <https://github.com/lexiforest/curl_cffi/issues/345>
* `httpx.Response.request.headers` isn't updated with default
  `curl-impersonate` headers,
  which can be unexpected on `CurlTransport.default_headers=True`
  <https://github.com/lexiforest/curl_cffi/issues/368>

* `CurlTransport.cert` argument isn't compatible
  with (deprecated) `httpx._types.CertTypes` -
  impossible to pass password as third tuple element,
  `pathlib.Path` (instead of strings in `httpx._types.CertTypes`) is forced
