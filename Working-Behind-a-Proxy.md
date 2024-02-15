PTBs default networking backend `HTTPXRequest` comes with built-in support for proxies.
Note that the details below only apply to `HTTPXRequest`.
If you use a different implementation of `BaseRequest`, you'll have to configure proxies yourself.

# How is a Proxy Server Chosen?

PTB will obtain its proxy configuration in the following order (the first to be found will be used):
1. Programmatic.
2. Using `HTTP_PROXY` environment variable.
3. Using `HTTPS_PROXY` environment variable.
4. Using `ALL_PROXY` environment variable.

# Setting a HTTP(S) Proxy Server Programmatically

Proxies can be setup like this:

```python
from telegram.ext import ApplicationBuilder

# "USERNAME:PASSWORD@" is optional, if you need authentication:
proxy_url = 'http://USERNAME:PASSWORD@PROXY_HOST:PROXY_PORT'  # can also be a https proxy
app = ApplicationBuilder().token("TOKEN").proxy(proxy_url).get_updates_proxy(proxy_url).build()
```

In the last line, we setup the proxy such that it'll be used both for making requests to the Bot API like  `Bot.send_message` ([`proxy()`](https://docs.python-telegram-bot.org/telegram.ext.applicationbuilder.html#telegram.ext.ApplicationBuilder.proxy)) and for fetching updates from Telegram ([`get_updates_proxy`](https://docs.python-telegram-bot.org/telegram.ext.applicationbuilder.html#telegram.ext.ApplicationBuilder.get_updates_proxy)). It is not necessary to setup a proxy for both, you can do it for either of them.

# Working Behind a Socks5 Server
This configuration is supported, but requires an optional/extra python package.
To install:
```bash
pip install python-telegram-bot[socks]
```
```python
from telegram.ext import ApplicationBuilder

proxy_url = "socks5://user:pass@host:port"

app = ApplicationBuilder().token("TOKEN").proxy(proxy_url).get_updates_proxy(proxy_url).build()
```

If you're more of an advanced user and would like to customize your proxy setup even further, check out the [docs of httpx](https://www.python-httpx.org/advanced/#http-proxying) for more info.