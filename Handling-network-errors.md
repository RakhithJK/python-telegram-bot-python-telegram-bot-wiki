# Background
Network errors can happen at many levels, on the local side, on the remote or somewhere along the way.

It is hard for applications to identify when an error occurs as it is not always something that can be identified in real time but rather when an additional event happens.

Network errors are a problem which each application have to deal with. Each application for its own use case and desired service level. 

# Where things can break?
## Locally (in PTB)
For something to "break" locally we need to close the network socket. This is something that PTB doesn't do voluntarily and basically if you've encountered such a case, it's a bug.

## Remotely on Telegram servers
There is a suspicion that Telegram are closing connections on some conditions. Whether it is true or not we can't really tell. The application should still be able to deal with it.

## Somewhere along the way
* A flaky network connection can cause connections to seem alive, although they are useless.
* Datacenters or ISPs which cut (probably inactive) network connections.
  * Azure drops inactive connections after 4 minutes by default (https://azure.microsoft.com/en-us/blog/new-configurable-idle-timeout-for-azure-load-balancer/).

# How to mitigate?

## Analysis
There is no one way to handle networking issues. Each bot has its own usage pattern and unique network behaviour.

Some bots will require providing answer to user requests within a limited time, while others can take all the time they need.
Some bots will be ok with "losing" a sent message once in a while, while others must ensure that the message was sent within a limited time frame.
Some bots send big files, while others only receive small files. Etc., etc.

In addition, the network connection takes an important factor. Hosting your bot in a region with poor Internet (bandwidth / packet lost / disconnections) is the obvious example, but it is not the only problem.
In case the bot is hosted "far" from the Telegram Bot servers you'll experience greater latency and possibly more packet loss. TCP should be able to handle packet loss, however, there are side effects for that (when will TCP identify the loss of a packet? what will be the window size? etc.).

Hosting the bot on a slow server can also affect performance and may be observed as networking issues.

Once you've understood the usage pattern of your bot (that includes the QoS) you can go into fine-tuning the ptb library for your needs.

## Tweaking PTB
### Networking backend tweaks

PTB performs HTTPS requests using the `telegram.request.BaseRequest` interface.
The method `BaseRequest.do_request` accepts four parameters for controlling timeouts:

* `read_timeout` specifies the maximum amount of time (in seconds) to wait for a response from Telegram’s server
* `write_timeout` specifies the maximum amount of time (in seconds) to wait for a write operation to complete (in terms of a network socket; i.e. POSTing a request or uploading a file)
* `connect_timeout` specifies the maximum amount of time (in seconds) to wait for a connection attempt to a Telegram server
* `pool_timeout` specifies the maximum amount of time (in seconds) to wait for a connection to become available from the connection pool

The built-in `request.HTTPXRequest` (which implements `BaseRequest`) uses a default of 5 seconds for `{read, write, connect}_timeout` and 1 second for `pool_timeout`.
`urllib3`.

The `write_timeout` is overwritten to 20 seconds when using send methods which attach files (`send_audio`, `send_document`, etc.).
When sending big files, calling `Bot.send_*(write_timeout=BIGGER_VALUE)` might be a good idea.

When using the standard `HTTPXRequest`, changing the defaults of `{read, write, connect, pool}_timeout` & `connect_timeout` is done when initializing the `Application`.

```python
Application.builder().token("Token").read_timeout(7).get_updates_read_timeout(42).build()
```

Note that there is both `ApplicationBuilder.read_timeout()` and `ApplicationBuilder.get_updates_read_timeout` (and similarly for the other timeouts) since PTB uses two different request objects for `Bot.get_updates` and all other bot methods.

See also the wiki page on the [[builder pattern|Builder-Pattern]].

### `Bot.get_updates`
When using `Application.run_polling()`/`Updater.start_polling()`, `getUpdates` is achieved using [Long Polling](https://en.wikipedia.org/wiki/Push_technology#Long_polling).
This means that the `get_updates` request is kept alive for  a [(Telegram defined) timeout](https://core.telegram.org/bots/api#getupdates) of X seconds to respond with new `Update`(s).
Only if updates are available before the X seconds have passed, the request is closed right away and `get_updates` is called again.
This also means that if a network error occurs during that timeout, it will be discovered only after the timeout has passed.

To be specific, when using the raw API (i.e. using `Bot` directly), the value you pass for `timeout` is added to the value of `read_timeout` and that is the exact timing when the error will be detected.
In that mode, the default value of `timeout` is 0 and the default value of `read_timeout` is 2.

If you're using `Updater.start_polling`/`Application.run_polling()` then the default value of `timeout` is overridden to 10. That means that the average time to identify network error is 6 seconds. 

Depending on the SLA you'd like to provide, if you suffer from frequent errors during `getUpdates` consider lowering the `timeout` value to catch the errors faster.
Please take in mind that too short polling intervals may have undesirable side effects:
* Telegram servers might conceive it as abusive and block your bot.
* Increase the load on your server and network connection.

### Socket options
Depending on your OS you may be able to set socket options to perform low level tweaks.
In the default networking backend, [`HTTPXRequest`](https://docs.python-telegram-bot.org/telegram.request.httpxrequest.html), there is a corresponding parameter for this.

## Stabilizing your app

Usually, when a network error occurs the exceptions will be printed to the log, and the app will recover (continue operation) after the connection has been restored. For smoother operation better suited to your use case, you may want to catch these exceptions and handle them yourself.
For this, be prepared to catch the [raised exception](https://python-telegram-bot.readthedocs.io/telegram.error.html) and handle it according to your policy (do you want to retry? ignore? other?) or use PTBs built-in mechanism for [[exception handling|Exceptions,-Warnings-and-Logging]].

This can involve a try-catch-block around send operations, catching `telegram.error.NetworkError`.
For catching network errors that occur during `get_updates()` (i.e.: during polling the telegram server), this can be done as shown in following example:

When using the `Application.builder()` framework, you can register an `error_callback` function as a part of the call to `application.updater.start_polling()`. This will cause the network error exceptions to be fed into the error handler function that has been registered with the `add_error_handler()` call.
```
def error_callback(error, application):
	application.create_task(application.process_error(update=None, error=error))

async def botloop() -> NoReturn:
	application = Application.builder().token(bot_token).build()
	application.add_handler(CommandHandler("start", start))
	application.add_error_handler(error_handler)
        # Application.run_polling() does not expose the error_callback parameter, so we need to
        # manually handle the startup and shutdown (usually all done by Application.run_polling() )
	await application.initialize()
	await application.updater.start_polling(allowed_updates=Update.ALL_TYPES, \
		error_callback=functools.partial(error_callback, application=application))
	await application.start()
	
	while True:
		await asyncio.sleep(1)
	
	# dropped from the loop -> exit. shutdown bot.
	await application.Updater.stop()
	await application.stop()
	await application.shutdown()
```
For more details see [Posting 3430](https://github.com/python-telegram-bot/python-telegram-bot/issues/3430) and the error handler code in [errorhandlerbot.py](https://docs.python-telegram-bot.org/en/stable/examples.errorhandlerbot.html)

## PTB
If you think of another way to improve stability from within ptb, please contact us (the maintainers).