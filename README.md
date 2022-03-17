[![Python package](https://github.com/xepor/xepor/actions/workflows/python-package.yml/badge.svg)](https://github.com/xepor/xepor/actions/workflows/python-package.yml)
[![PyPI-Server](https://img.shields.io/pypi/v/xepor.svg)](https://pypi.org/project/xepor/)
![PyPI - Status](https://img.shields.io/pypi/status/xepor)
[![Documentation Status](https://readthedocs.org/projects/xepor/badge/?version=latest)](https://xepor.readthedocs.io/en/latest/?badge=latest)
[![Project generated with PyScaffold](https://img.shields.io/badge/-PyScaffold-005CA0?logo=pyscaffold)](https://pyscaffold.org/)
[![996.icu](https://img.shields.io/badge/link-996.icu-red.svg)](https://996.icu)

# Xepor

[Xepor](https://github.com/xepor/xepor), a web routing framework for reverse engineers and security researchers.

## Features

1. Code everything with `@api.route()`, just like Flask! Write everything in *one* script and no `if..else` any more.
2. Handle multiple URL routes, even multiple hosts in one `InterceptedAPI` instance.
3. For each route, you can choose to modify the request *before* connecting to server (or even return a fake response without connection to upstream), or modify the response *before* forwarding to user.
4. Blacklist mode or whitelist mode. Only allow URL endpoints defined in scripts to connect to upstream, blocking everything else (in specific domain) with HTTP 404. Suitable for transparent proxying.
5. Human readable URL path definition and matching powered by [parse](https://pypi.org/project/parse/)
6. Host remapping. define rules to redirect to genuine upstream from your fake hosts. Regex matching is supported. **Best for SSL stripping and server side license cracking**!
7. Plus all the bests from [mitmproxy](https://github.com/mitmproxy/mitmproxy/)! **ALL** operation modes ( `mitmproxy` / `mitmweb` + `regular` / `transparent`  / `socks5` / `reverse:SPEC` / `upstream:SPEC`) are fully supported.

## Use Case

1. Evil AP and phishing through MITM.
2. Sniffing traffic from specific device by iptables + transparent proxy, modify the payload with xepor on the fly.
3. Cracking cloud based software license. See [examples/krisp/](https://github.com/xepor/xepor-examples/tree/main/krisp/) as an example.
4. ... and many more.

SSL stripping is NOT provided by this project.

# Quick start

```bash
mitmweb --web-host=\* --set connection_strategy=lazy -s example/httpbin/httpbin.py
```

Set your Browser HTTP Proxy to `<ip>:8080`, and access web interface at `<ip>:8081`, e.g. http://127.0.0.1:8081/

The `httpbin.py` do two things.

1. When user access http://httpbin.org/get, inject a query string parameter `payload=evil_param` inside HTTP request.
2. When user access http://httpbin.org/basic-auth/xx/xx/ (we just pretends we don't know the password), sniff `Authorization` headers from HTTP requests and print the password to the attacker.

Just what mitmproxy always do, but with code written in [*xepor*](https://github.com/xepor/xepor) way.

```python
from mitmproxy.http import HTTPFlow
from xepor import InterceptedAPI


HOST_HTTPBIN = "httpbin.org"

api = InterceptedAPI(HOST_HTTPBIN)


@api.route("/get")
def change_your_request(flow: HTTPFlow):
    """
    Modify URL query param.
    Test at:
    http://httpbin.org/#/HTTP_Methods/get_get
    """
    flow.request.query["payload"] = "evil_param"


@api.route("/basic-auth/{usr}/{pwd}", reqtype=InterceptedAPI.RESPONSE)
def capture_auth(flow: HTTPFlow, usr=None, pwd=None):
    """
    Sniffing password.
    Test at:
    http://httpbin.org/#/Auth/get_basic_auth__user___passwd_
    """
    print(
        f"auth @ {usr} + {pwd}:",
        f"Captured {'successful' if flow.response.status_code < 300 else 'unsuccessful'} login:",
        flow.request.headers.get("Authorization", ""),
    )


addons = [api]

```

